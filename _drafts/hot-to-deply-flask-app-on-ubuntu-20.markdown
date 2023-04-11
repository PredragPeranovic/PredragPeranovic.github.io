# How to deploy flask app on Ubuntu 20.04

Update server and create user

``` bash
# apt update
# apt upgrade
# adduser blokkod
# usermod -aG sudo blokkod
```

Install Nginx

``` bash
# apt install nginx
```

Configure letsencrypt.org

``` bash
curl https://get.acme.sh | sh -s email=my-email@acme.com
```

Close and reopen terminal to start using ``acme.sh``

Obtain and install certificate

``` bash
# acme.sh --issue -d taskclip.me -w /var/www/html/
# mkdir -p /etc/nginx/certs/server.acme.com
# acme.sh --install-cert -d server.acme.com  \
    --key-file /etc/nginx/certs/server.acme.com/key.pem \
    --fullchain-file /etc/nginx/certs/server.acme.com/cert.pem \
    --reloadcmd "service nginx force-reload"
```

> ```shell-session
> acme.sh will add cronjob to renew and reinstall certificates when they expires.
> ```

Import certificates in nginx

change ``/etc/nginx/sistes-enabled/default`` to

``` nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}
server {
        listen 443 default_server ssl http2;
        listen [::]:443 default_server ssl http2;

        server_name _;

        ssl_certificate /etc/nginx/certs/server.acme.com/cert.pem;
        ssl_certificate_key /etc/nginx/certs/server.acme.com/key.pem;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }
}
```

Enable firewall

``` bash
# ufw allow ssh
# ufw allow http
# ufw allow https
# ufw enable
```

## Deploy flask application

``` bash
$ sudo apt install python3.9 python3.9-dev python3.9-venv
```



``` bash
$ sudo mkdir /opt/acme
$ sudo chown acme.acme /opt/acme
$ cd /opt/acme
$ mkdir bin demoapp
```



``` bash
$ python3.9 -m venv /opt/acme/demoapp/.venv
```

Activate virtual env

```bash
$ cd /opt/acme/demoapp
$ . .venv/bin/activate
```

> Add copying of deployemet to server ssh .... shit

``` bash
(.venv) $ cd /opt/acme/demoapp
(.venv) $ pip install --upgrade pip
(.venv) $ pip install /home/acme/demoapp-0.1.0-py3-none-any.whl
(.venv) $ mkdir instance data
```



``` bash
$ vim instance/config.py
.... security key .. database redis etc connections
$ vim .flaskenv
FLASK_APP=demoapp
$ flask db upgrade
```





## Install postgres

``` bash
$ sudo apt install postgresql postgresql-server-dev-all gcc python3.9-dev python3-psycopg2
$ . /opt/acme/demoapp/.venv/bin/acitivate
(.venv) $ pip install psycopg2
(.venv) $ deactivate
$ sudo -sH
$ su - postgres
$ createuser acme
$ psql
```





``` sql
create database demo owner acme;
alter role acme set client_encoding to 'utf8';
alter role acme set default_transaction_isolation to 'read committed';
alter role acme set timezone to 'UTC';
```

## Install redis

``` bash
$ sudo apt install redis
```

## Install Gunicorn with eventlet

``` bash
(.venv) $ pip install gunicorn
(.venv) $ vim gunicorn.conf.py
import pathlib


wsgi_app = "demoapp:create_app()"

chdir = str(pathlib.Path(__file__).parent.absolute())

raw_env = [
    "FLASK_INSTANCE_PATH={}/instance".format(chdir),
]
```

### start gunicorn

`/etc/systemd/system/demoapp.socket`:

``` ini
[Unit]
Description=Demo App socket

[Socket]
ListenStream=/run/demoapp.sock

[Install]
WantedBy=sockets.target
```

`/etc/systemd/system/demoapp.service`:

```ini
[Unit]
Description=Demo App production service
Requires=demoapp.socket
After=network.target

[Service]
PermissionsStartOnly = true
User=acme
Group=acme
PIDFile=/run/demoapp/demoapp.pid
WorkingDirectory=/opt/acme/demoapp/
Environment=LANG=en_US.UTF-8
Environment=PATH=/opt/acme/demoapp/.venv/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
ExecStartPre=/bin/mkdir /run/demoapp
ExecStartPre=/bin/chown -R acme:acme /run/demoapp
ExecStart=/opt/acme/demoapp/.venv/bin/gunicorn \
	--user acme \
	--group acme \
	--config /opt/acme/demoapp/gunicorn.conf.py \
	--workers 4 \
	--log-level warn \
	--error-logfile /var/log/gunicorn/demoapp.log \
	--bind unix:/run/demoapp.sock \
	--pid /run/demoapp/demoapp.pid
PrivateTmp=true
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
ExecStopPost=/bin/rm -rf /run/demoapp
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## nginx to gunicorn

