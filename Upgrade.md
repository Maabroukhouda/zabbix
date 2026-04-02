# Zabbix Server Upgrade Procedure (6.4 → 7.0)

**Create a VM Clone and Change Network Configuration (Recommended)** 

Before starting the upgrade, it is strongly recommended to create a full clone of the Zabbix server VM and update its network configuration (IP address) to avoid conflicts with the production environment and allow safe testing.

**Objective**

Upgrade Zabbix Server from version 6.4 to 7.0, including:

* PostgreSQL upgrade
* Database cleanup
* OS upgrade
* Zabbix upgrade 

## 1. Database Analysis & Cleanup

Connect to PostgreSQL:
`sudo -i -u postgres psql`

* List databases and sizes

```
-- Liste des bases et leur taille
SELECT
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM
    pg_database
ORDER BY
    pg_database_size(pg_database.datname) DESC;
```


* Identify largest tables  
```
\c zabbix

SELECT
    schemaname,
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM
    pg_catalog.pg_statio_user_tables
ORDER BY
    pg_total_relation_size(relid) DESC
LIMIT 20;
```

* Cleanup Old Data
Remove history older than 3 months

```
DELETE FROM history WHERE clock < extract(epoch from now() - interval '3 months');
DELETE FROM history_uint WHERE clock < extract(epoch from now() - interval '3 months');
DELETE FROM history_str WHERE clock < extract(epoch from now() - interval '3 months');
```

* Remove trends older than 1 year
```
-- Supprimer les tendances plus anciennes
DELETE FROM trends WHERE clock < extract(epoch from now() - interval '1 year');
DELETE FROM trends_uint WHERE clock < extract(epoch from now() - interval '1 year');
```

* Optimize database
`VACUUM FULL;`

## 2. PostgreSQL Backup

```
ssh postgres@IP
 pg_dumpall -f ~/pg13_full_backup_$(date +%Y%m%d).sql
```
## 3. PostgreSQL Upgrade (13 → 17 )
* Configure PostgreSQL 17

`sudo nano /etc/postgresql/17/main/postgresql.conf`

Modify:

```
listen_addresses = '*'
                                      
port = 5432
```
* Stop/start clusters
```
sudo pg_ctlcluster 13 main stop
sudo pg_ctlcluster 17 main start
```
![image](/assets/pglist1.png)

* Restore backup

`sudo -u postgres psql -p 5432 -f /var/lib/postgresql/pg13_full_backup_20260325.sql postgres`

* Remove old version

```
sudo pg_dropcluster 13 main --stop
sudo apt remove --purge postgresql-13 postgresql-client-13 -y
sudo apt autoremove -y
```
![image](/assets/pglist2.png)

`ALTER USER zabbix WITH PASSWORD 'zabbix';`
## 4. Upgrade OS

```
sudo apt update && sudo apt upgrade -y 
sudo apt dist-upgrade -y
sudo apt autoremove -y 
sudo apt clean
sudo do-release-upgrade
```
## 5. Zabbix 7.0 Upgrade
* Add Zabbix 7 repository
```
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu24.04_all.deb
sudo apt update  
```
* Upgrade packages

`sudo apt install --only-upgrade zabbix-server-pgsql zabbix-frontend-php zabbix-agent`

## 6. Zabbix Configuration

`nano /etc/zabbix/zabbix_server.conf`
Add 

```
AllowUnsupportedDBVersions=1
$DB['SERVER']   = 'localhost';
$DB['PORT']     = '5432';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = 'zabbix';
```
## 7. Nginx Configuration
```
nano /etc/zabbix/nginx.conf

server {
    listen 8080;
    server_name IP_addresse;

    location ~ [^/]\.php(/|$) {
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_index index.php;
    }
}
```

## 8. Restart Services
```
systemctl restart postgresql
systemctl restart zabbix-server
systemctl restart nginx
systemctl restart php8.1-fpm
```