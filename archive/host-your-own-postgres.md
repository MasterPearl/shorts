# Host Your Own Postgres

:elephant: Get running with the last version of Postgres in minutes

## Set Up Server

Spin up a new server with Ubuntu 16.04.

Firewall

```sh
sudo ufw allow ssh
sudo ufw enable
```

[Automatic updates](https://help.ubuntu.com/16.04/serverguide/automatic-updates.html)

```sh
sudo apt-get -y install unattended-upgrades
echo 'APT::Periodic::Unattended-Upgrade "1";' >> /etc/apt/apt.conf.d/10periodic
```

Time zone

```sh
sudo dpkg-reconfigure tzdata
```

and select `None of the above`, then `UTC`.

## Install Postgres

Install PostgreSQL 10

```sh
echo "deb https://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install -qq -y postgresql-10 postgresql-contrib
```

## Configure

Edit `/etc/postgresql/10/main/postgresql.conf`.

```sh
# general
max_connections = 100

# logging
log_min_duration_statement = 100 # log queries over 100ms
log_temp_files = 0               # log all temp files

# stats
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 1000
```

## Remote Connections

Enable remote connections if needed

```sh
echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/9.6/main/pg_hba.conf
echo "listen_addresses = '*'" >> /etc/postgresql/9.6/main/postgresql.conf
sudo service postgresql restart
```

And update the firewall

```sh
sudo ufw allow 5432/tcp # for all ips
sudo ufw allow from 127.0.0.1 to any port 5432 proto tcp # specific ip
sudo ufw enable
```

## Provisioning

Create a new user and database for each of your apps

```sh
sudo su - postgres
psql
```

And run:

```sql
CREATE USER myapp WITH PASSWORD 'mypassword';
ALTER USER myapp WITH CONNECTION LIMIT 20;
CREATE DATABASE myapp_production OWNER myapp;
```

Generate a random password with:

```sh
cat /dev/urandom | LC_CTYPE=C tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

## Backups

### Daily

Store backups on S3

- [Amazon S3 Backup Scripts](https://github.com/collegeplus/s3-shell-backups/blob/master/s3-postgresql-backup.sh)
- [Automatic Backups to Amazon S3 are Easy ](https://rossta.net/blog/automatic-backups-to-amazon-s3-are-easy.html)

*TODO: better instructions*

### Continuous

Rollback to a specific point in time with [WAL-E](https://github.com/wal-e/wal-e).

Opbeat has a [great tutorial](https://opbeat.com/blog/posts/postgresql-backup-to-s3-part-one/).

## Logging

[Papertrail](https://papertrailapp.com) is great and has a free plan.

Install remote syslog

```sh
cd /tmp
wget https://github.com/papertrail/remote_syslog2/releases/download/v0.13/remote_syslog_linux_amd64.tar.gz
tar xzf ./remote_syslog*.tar.gz
cd remote_syslog
sudo cp ./remote_syslog /usr/local/bin
```

Create `/etc/log_files.yml` with:

```sh
files:
  - /var/log/postgresql/*.log
destination:
  host: logs.papertrailapp.com
  port: 12345
  protocol: tls
```

### Archive

Archive logs to S3

```sh
sudo apt-get install logrotate s3cmd
s3cmd --configure
```

Add to `/etc/logrotate.d/postgresql-common`:

```conf
sharedscripts
postrotate
  s3cmd sync /var/log/postgresql/*.gz s3://mybucket/logs/
endscript
```

Test with:

```sh
logrotate -fv /etc/logrotate.d/postgresql-common
```

## TODO

- scripts

  ```sh
  pghost bootstrap
  pghost allow all
  pghost allow 127.0.0.1
  pghost backup:all
  pghost backup myapp
  pghost restore myapp
  pghost provision myapp
  pghost logs:syslog logs.papertrailapp.com 12345
  pghost logs:archive mybucket/logs
  ```

- monitoring (Graphite, CloudWatch, etc)

## Resources

- [Copy your server logs to Amazon S3 using Logrotate and s3cmd](https://www.shanestillwell.com/2013/04/04/copy-your-server-logs-to-amazon-s3-using-logrotate-and-s3cmd/)
