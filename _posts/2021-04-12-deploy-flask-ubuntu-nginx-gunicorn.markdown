---
layout: post
title:  "How to deploy Flask app on Ubuntu 20.04 with Nginx and Gunicorn"
date:   2021-04-12 23:30:00 +0100
categories: ["Writing challenge", "How to"]
---

Lets define **minimum conditions of acceptable deployment** as:

- Application sits behind “proper” web server in this case **Nginx**;
- Application runs on ”proper” application server in this case **Gunicorn**;
- Application startup or shutdown is managed by native Ubuntu service manager in this case **systemd**;
- Application data, configuration and install folders are separated from each other so thay can be independently backuped and restored. 
- Application upgrade is done by executing **pip install --upgrade**.

## Create Flask Demo Application

We need a Flask application to deploy, lets build the "simplest" one. I’ll develop app on an Ubuntu machine with Python 3.8 and [Poetry](https://python-poetry.org/), you can use whatever you are familiar with.

> In case you are developing on Ubuntu 18.04/20.04 you can prepare development enviroment with:
> ```
> $ sudo apt install python3.8 python3.8-dev python3.8-venv
> $ curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python3.8 - 
> ```
>

Create demoapp application directory and initialise Python virtual enviroment (venv):

{% highlight shell-session %}
$ mkdir ~/demoapp
$ cd ~/demoapp
$ poetry init
{% endhighlight %}

When executing `poetry init` accept all defaults answers except for questions about **defining dependencies interactively** where you should choose **“no”**. 
Poetry will create venv inside `~/.cache/pypoetry/virtualenvs/demoapp-xxxx` just for this application. That you can activate by executing `poetry shell` inside `~/demoapp`.

Activate python venv and create demoapp directory structure:

{% highlight shell-session %}
$ cd ~/demoapp
$ poetry shell
(demoapp-xx) $ poetry add flask
(demoapp-xx) $ mkdir demoapp
(demoapp-xx) $ mkdir demoapp/templates
{% endhighlight %}


> `(demoapp-xx) $` in front of prompt indicates that python venv is active.

Add following lines inside `demoapp/pyproject.toml` **at end of [tool.poetry]** section just before *[tool.poetry.dependencies]*:

{% highlight ini %}
packages = [
    { include = "demoapp" }
]
{% endhighlight %}

Create `demoapp/__init__.py`:
{% highlight python %}
import os
from flask import Flask, render_template


def create_app(config=None):
    instance_path = os.environ.get("FLASK_INSTANCE_PATH", None)
    app = Flask(
        __name__, 
        instance_relative_config=True, 
        instance_path=instance_path
    )
    if config is not None:
        if isinstance(config, dict):
            app.config.from_mapping(config)
        elif config.endswith(".py"):
            app.config.from_pyfile(config)
    else:
        app.config.from_pyfile("config.py", silent=True)
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    @app.route("/")
    def index():
        """Servers intex.html"""
        return render_template("index.html")

    return app
{% endhighlight %}

Create `demoapp/templates/index.html`:

{% highlight html%}
<html>
<body>
    Hello from Demo application.
</body>
</html>
{% endhighlight %}

You will notice that I'm preparing this app to be packaged for easy deployment by setting up **instance_path** and **instance_relative_config** to Flask and by adding **packages** directive in **pyproject.toml**.

To test your app run inside `~/demoapp`

{% highlight shell-session %}
(demoapp-xx) $ export FLASK_APP=demoapp
(demoapp-xx) $ flask run
{% endhighlight %}

Open up http://localhost:5000 in the browser and you should see page displaying "Hello from Demo application".

### Prepare "demoapp" Package for Deployment

Create **demoapp-0.1.0-py3-none-any.whl** and **demoapp-0.1.0.tar.gz** distribution packages inside the `~/demoapp/dist`:

{% highlight shell-session %}
(demoapp-xx) $ poetry build
{% endhighlight %}


## Configuring Ubuntu 20.04

I’ll assume following:

- You have root privilegies on fresh instalation of Ubuntu 20.04;
- **Acme company** has built this “demoapp”;
- You’re creating an “**acme**” user to be responsible for managing demoapp;
- You’re hosting the app on a public server named **server.acme.com**;
- You know how to setup domain name to point on your server

### Add User Responsible for Managing Application

Upgrade system, add **acme** user and allow acme to sudo as root:

{% highlight shell-session %}
# apt update
# apt upgrade
# adduser acme
# usermod -aG sudo acme
{% endhighlight %}

### Install Nginx

{% highlight shell-session %}
# apt install nginx
{% endhighlight %}

Go to *http://server.acme.com* to check if website is running.

> If browser is constantly trying to redirect you to https://server.acme.com use
> browser incognito/private mode to access website over http.

### Configure Nginx to Use HTTPS With "letsencrypt.org" Certificate

I’ll use [acme.sh](https://github.com/acmesh-official/acme.sh) script to obtain and renew certificates. Install it with:

{% highlight shell-session %}
#   curl https://get.acme.sh | sh -s email=my-email@acme.com
{% endhighlight %}

Close and reopen your terminal to start using acme.sh.

#### Obtain and install certificate:

{% highlight shell-session %}
# acme.sh --issue -d server.acme.com  -w /var/www/html/
# mkdir -p /etc/nginx/certs/server.acme.com
# acme.sh --install-cert -d server.acme.com  \
    --key-file /etc/nginx/certs/server.acme.com/key.pem \
    --fullchain-file /etc/nginx/certs/server.acme.com/cert.pem \
    --reloadcmd "service nginx force-reload"
{% endhighlight %}

> acme.sh will add cronjob to renew and reinstall certificates when thay expires.


#### Import Certificates in Nginx 

Change `/etc/nginx/sites-enabled/default` to:

{% highlight nginx %}
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

        ssl_certificate /etc/nginx/certs/acme.jembe.io/cert.pem;
        ssl_certificate_key /etc/nginx/certs/acme.jembe.io/key.pem;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

Now every HTTP request will be redirected to HTTPS. This is good for now but we'll change this after installing our demoapp.

### Enable Firewall

Use the simplest firewall configuration as:

{% highlight shell-session %}
# ufw allow ssh
# ufw allow http
# ufw allow https
# ufw enabled
{% endhighlight %}

Check firewall status with `# ufw status`.

## Deploy Flask Application

We'll install our Flask "demoapp" under `/opt/acme` as user **acme**. 

### Preaparing venv and Directory Structure

Install appropriate python version (in this case Python 3.8):

{% highlight shell-session %}
$ sudo apt install python3.8 python3.8-dev python3.8-venv
{% endhighlight %}

Create deployment direcotry structure:

{% highlight shell-session %}
$ sudo mkdir /opt/acme
$ sudo chown acme.acme /opt/acme
$ cd /opt/acme
$ mkdir bin envs demoapp
{% endhighlight %}

Following dirs are created for:

- **/opt/acme/bin**: all bash scripts realated to our demoapp;
- **/opt/acme/envs**: python virtual enviroments;
- **/opt/acme/demoapp**: application configuration and data files;

Create venv:

{% highlight shell-session %}
$ python3.8 -m venv /opt/acme/envs/demoapp
{% endhighlight %}

Activate venv:

{% highlight shell-session %}
$ source /opt/acme/envs/demoapp/bin/activate
{% endhighlight %}

#### Activate venv with .bash_aliases

If you are lazy as me, you can add following line in `~/.bash_aliases`, to activate venv by typing `demoapp`:

{% highlight shell-session %}
alias demoapp='export PS1="\u:\W\$ ";source /opt/acme/envs/demoapp/bin/activate;cd /opt/acme/demoapp'
{% endhighlight %}

After you close and reopen your terminal you can type `demoapp` to activate python venv
and go to application directory.

### Install "demoapp"

{% highlight shell-session %}
(demoapp) $ cd /opt/acme/demoapp
(demoapp) $ pip install --upgrade pip
(demoapp) $ pip install /home/acme/demoapp-0.1.0-py3-none-any.whl
(demoapp) $ mkdir instance data
{% endhighlight %}

> `(demoapp) $` indicates that you should run following commands in python venv for demoapp.
> I'm also assuming that you already copied package with flask demoapp in `/home/acme`.

> Directory `/opt/acme/demoapp/instance` is [flask instance folder](https://flask.palletsprojects.com/en/1.1.x/config/#instance-folders){:target="_blank"}.

> I usually use `/opt/acme/demoapp/data`  to save all app data that need to be saved directly on hard disk. 

Create `/opt/acme/demoapp/instance/config.py`:

{% highlight python %}
SECRET_KEY = "put some random secret key here"
{% endhighlight %}

You can generate random secret key with 

{% highlight shell-session %}
$ python3.8 -c "import os; print(os.urandom(24))"
{% endhighlight %}

## Install and Configure Gunicorn

{% highlight shell-session %}
(demoapp) $ pip install gunicorn
{% endhighlight %}

Create `/opt/acme/demoapp/gunicorn.conf.py`:

{% highlight python %}
import pathlib


wsgi_app = "demoapp:create_app()"

chdir = str(pathlib.Path(__file__).parent.absolute())

raw_env = [
    "FLASK_INSTANCE_PATH={}/instance".format(chdir),
]
{% endhighlight %}

Settings in `gunicorn.conf.py` file will tell gunicorn how to start our demoapp and also will tell demoapp where it is its INSTANCE_PATH directory that contains application configuration.

### Create systemd Scripts for Starting and Stoping Gunicorn

`/etc/systemd/system/demoapp.socket`:


{% highlight ini %}
[Unit]
Description=Demo App socket    

[Socket]
ListenStream=/run/demoapp.sock

[Install]
WantedBy=sockets.target
{% endhighlight %}


`/etc/systemd/system/demoapp.service`:

{% highlight ini %}
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
Environment=PATH=/opt/acme/envs/demoapp/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
ExecStartPre=/bin/mkdir /run/demoapp
ExecStartPre=/bin/chown -R acme:acme /run/demoapp
ExecStart=/opt/acme/envs/demoapp/bin/gunicorn \
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
{% endhighlight %}


{% highlight shell-session %}
$ sudo mkdir /var/log/gunicorn
$ sudo chown acme.acme /var/log/gunicorn
$ sudo systemctl enable demoapp.service
$ sudo systemctl enable demoapp.socket
$
$ sudo systemctl start demoapp.socket
{% endhighlight %}

You can use `sudo systemctl status|start|stop|enable|disable demoapp.service` commands to manage or check status of demoapp services.

### Configure Nginx to Proxy Requests to Gunicorn

Open `/etc/nginx/sites-enabled/default` configured previously and change `location /` directive to:


{% highlight nginx %}
	location / {
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_redirect off;
		proxy_pass http://unix:/run/demoapp.sock;
		access_log  /var/log/nginx/demoapp.access.log;
		error_log  /var/log/nginx/demoapp.error.log;
	}
{% endhighlight %}

{% highlight shell-session %}
$ sudo systemctl restart nginx
{% endhighlight %}

## Serve Flask App with URL Prefix

Open `/opt/acme/demoapp/instance/config.py` and add line:

{% highlight python %}
APPLICATION_ROOT = “/demoapp/”
{% endhighlight %}


Open `/etc/nginx/sites-enabled/default` and change `location /` directive to:

{% highlight nginx %}
	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}
	location /demoapp {
		proxy_set_header SCRIPT_NAME /demoapp;

		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_redirect off;
		proxy_pass http://unix:/run/demoapp.sock;
		access_log  /var/log/nginx/demoapp.access.log;
		error_log  /var/log/nginx/demoapp.error.log;
	}
{% endhighlight %}

## How to Upgrade App

Copy new version of `demoapp` into `/home/acme` and login to server:

{% highlight shell-session %}
$ demoapp
(demoapp) $ pip remove demoapp
(demoapp) $ pip install /home/acme/demoapp-0.2.0-py3-none-any-whl
(demoapp) $ sudo systemctl restart demoapp.service
{% endhighlight %}

When you are installing or upgrading "real" Flask app that uses database, saves data on
local storages and access other system or network resources you will probably need to execute
additional commands like `flask migrate` etc.

## Feedback 

If you have any feedback or you want to see more posts like this, let me know on [Twitter](https://twitter.com/peranp){:target="_blank"}.