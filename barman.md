*** Postgres server : 192.168.4.14 ***
*** Barman Server : 12.168.4.13 ***


# barman Side
# barman Backup Strategy 

1- Wall Streaming
![image](https://user-images.githubusercontent.com/88557305/209461497-918d6303-fbd8-4d21-9de5-d9b4d1eaf63f.png)

2- Rsync 
![image](https://user-images.githubusercontent.com/88557305/209461486-3915c994-24dd-49dd-97ff-083125c27303.png)


1- instalation
```
sudo apt-get install barman
```
2-  make a user for barman and specify the passsword that barman user 

```
sudo passwd barman
sudo -i -u barman
echo "1921.68.4.14:5432:*:barman:password" >> ~/.pgpass
```

3- Postgres server SSH Key config 
either postgres should has access to barman or barman
so we copy barman ssh key in postgres server
 
```
ssh-keygen -t rsa
ssh-copy-id postgres@192.168.4.210
```

4- make config for barman 

vim /etc/barman.d/pgsql.conf

```
[pgsql]
description =  "Postgres server"
conninfo = host=192.168.4.210 user=barman dbname=postgres
archiver=on
backup_method = rsync
ssh_command = ssh postgres@192.168.4.210
retention_policy = RECOVERY WINDOW OF 2 WEEKS
```

5- Verify Barman conectivity
```
 barman check pgsql
```
5-1 if you dont have a Enough data run this command to archive wall (this should be done before barman check)
```
barman switch-xlog --force --archive pgsql 
```
![image](https://user-images.githubusercontent.com/88557305/206809977-518b0953-2889-4986-98a0-39b217d38ad6.png)

6- take backup 
```
barman backup pgsql
```
![image](https://user-images.githubusercontent.com/88557305/206810138-151853b6-1f22-470a-90ce-8ce0a535862c.png)


7- barman commands
```
barman show-server pgsql | grep incoming_wals_directory
 barman list-backup pgsql (servername which defiend in barman.conf)
barman show-backup pgsql backup_id
```
![image](https://user-images.githubusercontent.com/88557305/206810281-92781b77-6850-4577-b0ca-6ef4438a863a.png)


8- recover to specified time 
we can see the backup time in barman show-backup pgsql backup_id
```
 barman recover --remote-ssh-command "ssh postgres@192.168.4.210"  --target-time="2022-12-09 22:02:39.970740+00:00
" pgsql  20221209T220233  /var/lib/postgresql/14/data

```
![image](https://user-images.githubusercontent.com/88557305/206809875-0828d913-48df-4047-b13b-6022baa9756d.png)

*** We can Schedule backup from crontab  ***
```
sudo -i -u barman
crontab -e
# To run a backup daily at 3:30, add the following line to the cron tab and save it:
30 03 * * * /usr/bin/barman backup pgsql
```

------------------------------------------------------------------------------------------

# Postgres server Side 

1- creating user barman for  postgres server & Linux 

```
adduser barman
sudo -i -u postgres
createuser --interactive -P barman

```

2- Config postgres to getting access from barman
vim /etc/postgresql/14/main/postgresql.conf
```
listen_addresses = 'localhost, 192.168.4.210'
```
```
sudo service postgresql restart
```
verify barman access
```
psql -c 'SELECT version()' -U barman -h localhost postgres
```

3- Copy barman server Key to posgres account on postgres server
```
su - postgres
ssh-keygen -t rsa
ssh-copy-id barman@192.168.4.25
ssh-copy-id postgres@localhost
ssh postgres@localhost -C true
```
4- Get access from barman to postgress on port 5432

Append this line to /etc/postgresql/14/main/pg_hba.conf
```
host    all             all             192.168.4.25/32         trust
```

5- Turn on archive mode and Specify Backup method and Rsync address
Postgres will send archive to barman

 vim /etc/postgresql/14/main/postgresql.conf
```
listen_addresses = '0.0.0.0' 
wal_level = archive
archive_mode = on
archive_command = 'rsync -a %p barman@192.168.4.25:/var/lib/barman/pgsql/incoming/%f'


```
sudo systemctl restart postgresql

*** Postgres wal directory  
/var/lib/postgresql/14/main/pg_wal/
***
![image](https://user-images.githubusercontent.com/88557305/206810056-4a00b831-0ba7-4c2b-953d-be6a6893550e.png)

# Barman Monitoring
```
pip3 install barman-exporter
```
vim /etc/systemd/system/barman-exporter.service
```
[Unit]
Description=Barman Exporter
After=network-online.target

[Service]
Type=simple
User=barman
Group=barman
ExecStart=/usr/local/bin/barman-exporter  -l 192.168.4.25:9780 pgsql
SyslogIdentifier=barman_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```
in prometheus config 
```
  - job_name: 'barman'
    metrics_path:
    static_configs:
      - targets:
        - 192.168.4.25:9780

```


![image](https://user-images.githubusercontent.com/88557305/208292440-23a73a4f-ef44-473c-9ab9-1fd236699eb4.png)
