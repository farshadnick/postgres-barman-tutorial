**Postgres server : 192.168.4.210 **
** Barman Server : 12.168.4.25 **


# barman Side

1- instalation
```
sudo apt-get install barman
```
2-  make a user for barman and specify the passsword that barman user 

```
sudo passwd barman
sudo -i -u barman
echo "pgsql:5432:*:barman:password" >> ~/.pgpass
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

[pgsql]
description =  "Postgres server"
conninfo = host=192.168.4.210 user=barman dbname=postgres
ssh_command = ssh postgres@192.168.4.210
retention_policy = RECOVERY WINDOW OF 2 WEEKS



 **Recover Backup from Barman**

 barman recover --remote-ssh-command "ssh postgres@192.168.4.210"  --target-time="2022-12-09 22:02:39.970740+00:00
" pgsql  20221209T220233  /var/lib/postgresql/14/data

5- Verify Barman conectivity
```
 barman check pgsql
```
6- take backup 
```
barman backup pgsql
```
7- barman commands
```
 barman list-backup pgsql (servername which defiend in barman.conf)
barman show-backup pgsql backup_id


```
8- recover to specified time 
we can see the backup time in barman show-backup pgsql backup_id
```
 barman recover --remote-ssh-command "ssh postgres@192.168.4.210"  --target-time="2022-12-09 22:02:39.970740+00:00
" pgsql  20221209T220233  /var/lib/postgresql/14/data

```

# Postgres server Side 

1- creating user barman in postgres server

```
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

wal_level = archive
archive_mode = on
archive_command = 'rsync -a %p barman@192.168.4.25:/var/lib/barman/pgsql/incoming/%f'


```
sudo systemctl restart postgresql


