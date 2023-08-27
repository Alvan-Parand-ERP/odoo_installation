# Odoo Installation Guide

### 1. Python
Suggested python version: 3.8
``````
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.8 python3.8-venv python3.8-full python3.8-dev
``````


### 2. Postgresql
``````
sudo apt install postgresql
``````
```
sudo -i -u postgres
psql
> create user odoo with encrypted password 'odoo';
> alter user odoo with superuser;
> alter user odoo with createdb;
```

### 3. Ubuntu packages
```
sudo apt install wkhtmltopdf nodejs npm less
```

### 4. Js modules
```
sudo npm install -g rtlcss
sudo npm install -g less
sudo npm install -g less-plugin-clean-css
```

### 5. Create and use virtualenv
```
python3.8 -m venv venv
source venv/bin/activate
```

### 6. Install python dependencies on venv
##### 6.1. Note
```
* psycopg2 -> psycopg2-binary
* Werkzeug==0.16.1 ; python_version <= '3.9'
* Werkzeug==2.0.2 ; python_version > '3.9'
* python-ldap -> python3-ldap
- remove version of `python3-ldap`
- remove version of `lxml` package
+ pyopenssl==22.0.0
+ cryptography==37.0.0

```
#### 6.2. next install packages
```
pip install wheel setuptools
pip install -r requirements.txt
```

### 7. Create ```odoo.conf``` file
```
[options]
addons_path = ***
admin_passwd = ***
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo
dbfilter = 
data_dir = ***
http_port = 8069
longpolling_port = 8072
logfile = ***

```

### 8. Create odoo service file
#### /etc/systemd/system/odoo.service
```
[Unit]
Description=Odoo
Requires=postgresql.service
After=network.target postgresql.service
[Service]
Type=simple
SyslogIdentifier=odoo
PermissionsStartOnly=true
User=root
Group=root
ExecStart= /opt/odoo/venv/bin/python3 /opt/odoo/odoo-bin -c /opt/odoo/odoo.conf
StandardOutput=journal+console
[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl start odoo
sudo systemctl enable odoo
```

### 9. Nginx configuration
first add ```proxy_mode = True``` to odoo.conf file

#### /etc/nginx/sites-enabled/odoo.conf
```
#odoo server
upstream odoo {
  server 127.0.0.1:8069;
}
upstream odoochat {
  server 127.0.0.1:8072;
}

# http -> https
server {
  listen 80;
  server_name odoo.mycompany.com;
  rewrite ^(.*) https://$host$1 permanent;
}

server {
  listen 443 ssl;
  server_name odoo.mycompany.com;
  proxy_read_timeout 720s;
  proxy_connect_timeout 720s;
  proxy_send_timeout 720s;

  # Add Headers for odoo proxy mode
  proxy_set_header X-Forwarded-Host $host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Real-IP $remote_addr;

  # SSL parameters
  ssl_certificate /etc/ssl/nginx/server.crt;
  ssl_certificate_key /etc/ssl/nginx/server.key;
  ssl_session_timeout 30m;
  ssl_protocols TLSv1.2;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;

  # log
  access_log /var/log/nginx/odoo.access.log;
  error_log /var/log/nginx/odoo.error.log;

  # Redirect longpoll requests to odoo longpolling port
  location /longpolling {
    proxy_pass http://odoochat;
  }

  # Redirect requests to odoo backend server
  location / {
    proxy_redirect off;
    proxy_pass http://odoo;
  }

  # common gzip
  gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;
  gzip on;
}
```

### 10. Live chat and video call configurations
this feature works with domain name and longpolling. long polling default port is 8072 and nginx comes in Nginx configuration part
### /etc/nginx/sites-enabled/odoo.conf
enable ssl and proxy_pass

### /etc/nginx/sites-enabled/odoo.conf
add this configrurations to ```odoo.conf```
<ul>
    <li>Server with 4 CPU, 8 Thread</li>
    <li>60 concurrent users</li>
    <li>60 users / 6 = 10 <- theoretical number of worker needed</li>
    <li>(4 * 2) + 1 = 9 <- theoretical maximal number of worker</li>
    <li>We’ll use 8 workers + 1 for cron. We’ll also use a monitoring system to measure cpu load, and check if it’s between 7 and 7.5 .</li>
    <li>RAM = 9 * ((0.8*150) + (0.2*1024)) ~= 3Go RAM for Odoo</li>
</ul>    

```
[options]
limit_memory_hard = 1677721600
limit_memory_soft = 629145600
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200
max_cron_threads = 1
workers = 8
proxy_mode = True
```