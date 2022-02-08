---
layout: post
title:  "How to Properly Organize Yours Flask Projects"
date:   2022-02-01 10:53:00 +0100
categories: ["Flask", "How to", "Jembe" ]
---

You have probably already read flask documentation about creating new projects at https://flask.palletsprojects.com/
and everytihing is nice and clear but now you are in front of your computer starting new project and don't know how and where to start.

So many choices on how to organize your aplication, packages to use etc, to the point that its become overvelming expecialy if you are relativly new to the framework.

Should you use template for minimal application or for "dynamic" application? Should you package your application in python package? So many decision to make and so many choices and your time is ticking you need to start coding to have your app ready for presentation.

This is step by step instructions on starting new Flask project, fast and with proper structure.

Disclamer, this is not the only proper structure thear are infinite number of good/proper/well orgnaized flask applications but we need one to start from.


# Install flask

Create Python Virtual Enviroment and activate it:

{% highlight shell-session %}
$ mkdir your-project
$ cd your-project
$ python -m venv .venv
$ . .venv/bin/activate
{% endhighlight %}

Now you want to install Flask but, don't do it!

First lets create project structure and configuration files like so:

{% highlight shell-session %}
    .flaskenv
    .gitignore
    LICENSE
    MANIFEST.in
    pyproject.toml
    README.md
    setup.cfg
    setup.py
    /your_project
        __init__.py
        app.py
        /models
            __init__.py
        /views
            __init__.py
        /templates
        /static
            main.css
    /data
    /instance
        config.py
    /tests
{% endhighlight %}

PLEASE DON'T CLOSE THE PAGE, here me out:

Not all of the files above are esential to start coding, but thay are all esential when you want to deploy your project and colaborate with others on development (or use two diferent development machine, one at home other at work :) )

content of this files is boiltape, create and foget, and it's easaer to create it now then to thing about them in the middle of your project when you have hudreds other chalages to tackle.

Lets crate all files above and explaint it breafly

So you can easyly run your app with `flask run` without thinking about enviroment variables ever again.

.flaskenv

{% highlight ini %}
FLASK_APP=your_project
FLASK_ENV=development
{% endhighlight %}


.gitignore
{% highlight ini %}
# It's long, but trust me i'm profesional :)
.DS_Store
/.env
/.venv
# /.flaskenv
*.pyc
*.pyo
env/
env*
/dist/
/build/
*.egg
*.egg-info/
_mailinglist
.tox/
.cache/
.mypy_cache
.pytest_cache/
.idea/
docs/_build/
.vscode

# Coverage reports
htmlcov/
.coverage
.coverage.*
*,cover

/build/
/data/

# javascript/node
/node_modules/
{% endhighlight %}

Everybody needs to licence their apps... so thay say.

 LICENSE

{% highlight ini %}
Your Project Copyright (C) 2022 Your Name your@email.address
# Go to https://opensource.org/ or put some propreatary license or leave it blank :)
{% endhighlight %}

We will use setuptools for packaging application and we need following files to do it right (just copy-paste code from it):

setup.py
{% highlight Python %}
from setuptools import setup
setup()
{% endhighlight %}

pyproject.toml
{% highlight ini %}
[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.coverage.run]
source = ["your_project"]
branch = true


[build-system]
requires = ["setuptools","wheel"]
build-backend = "setuptools.build_meta"
{% endhighlight %}

MANIFEST.in
{% highlight ini %}
recursive-include your_project *.py *.html *.css *.js *.jpt
prune .venv
prune node_modules
prune dist
prune data
prune instance
{% endhighlight %}

setup.cfg
{% highlight ini %}
[metadata]
name = your_project
version = 0.0.1
description = Your project
# Look at https://spdx.org/licenses/ for license indetifier or use "proprietary"
# license = MIT
license_files = LICENSE

# Enter your full name or the name of your company
author = Your name
# Enter your email or email of your company
# author_email = 

long_description = file: README.md

# Home page of the project
# url= https://yourproject.com

# Classifiers for project if you are going to upload it to pypi.org
# https://pypi.org/classifiers/
# classifiers =
#     Development Status :: 3 - Alpha
#     Environment :: Web Environment
#     Framework :: Flask
#     License :: OSI Approved :: GNU Affero General Public License v3 or later (AGPLv3+)
#     License :: Other/Proprietary License
#     Programming Language :: Python :: 3.8

# keywords = ["your_project"]

# project_urls=
#     Repository = https://github.com/YourName/YourProject
#     Documentation = https://github.com/YourName/YourProject

[options]
packages = find:
include_package_data = True
python_requires = >=3.8
install_requires = 
    python-dotenv
    Flask
    Flask-SQLAlchemy
    Flask-Migrate
    Flask-Babel
    Flask-SeaSurf
    Flask-Session
    Flask-Login
    Flask-Mail
    WTForms
    email-validator
    
[options.packages.find]
exclude = tests

[options.extras_require]
dev = 
    black
    mypy
    pytest
    coverage [toml]

[options.entry_points]
console_scripts = 
    your_project = your_project.cli.scripts:cli
{% endhighlight %}

Why setuptools? Why not poetry or something "easier" to use?
 - You want minimal dependency for implementation/deployment in production, you realy don't want to fight with poetry, or any other similar tool in production. 
 - You want same enviroment in production and in development. Exacly the same, same commands same file structure etc.. That means less things/commands to rember easaer to reproduce bugs in development, easear to debug on production server ;) (never debug programs on production big No-No).

 Downside of using setuptools is when you introduce new dependency to your project you must manualy edit setup.cfg install_requires.


 Now lets create application scelethon, 

/your_package/__init__.py
{% highlight python %}
import os
from flask import Flask
from . import app as app_services


def create_app(config=None) -> "Flask":
    from . import views, commands

    instance_path = os.environ.get(
        "FLASK_INSTANCE_PATH", os.path.join(os.getcwd(), "instance")
    )
    app = Flask(__name__, instance_relative_config=True, instance_path=instance_path)

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

    app_services.init_app(app)
    views.init_views(app)
    commands.init_commands(app)

    return app

{% endhighlight %}

/your_package/app.py
{% highlight python %}
from typing import TYPE_CHECKING
import babel
from flask_seasurf import SeaSurf
from flask_session import Session
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_babel import Babel
from flask_login import LoginManager
from flask_mail import Mail

if TYPE_CHECKING:
    from flask import Flask

__all__ = ("init_app", "jmb", "csrf", "session", "db", "migrate")

csrf = SeaSurf()
session = Session()

db = SQLAlchemy()
migrate = Migrate()

login_manager = LoginManager()
mail = Mail()

babel = Babel()


@login_manager.user_loader
def load_user(user_id):
    from conprob.models import User

    if user_id is not None:
        return db.session.query(User).get(user_id)
    return None


def init_app(app: "Flask"):
    with app.app_context():
        csrf.init_app(app)
        session.init_app(app)

        login_manager.init_app(app)

        db.init_app(app)
        migrate.init_app(app, db=db)

        mail.init_app(app)

        babel.init_app(app)


{% endhighlight %}

/your_project/views/__init__.py
{% highlight python %}
from typing import TYPE_CHECKING
from flask import redirect
from jembe import page_url

if TYPE_CHECKING:
    from flask import Flask


def init_views(app: "Flask"):
    """Flask regular routes"""

    @app.route("/")
    def index():
        """Redirects to Main jembe page."""
        return redirect(page_url("/main"))

{% endhighlight %}

/your_project/commands.py
{% highlight python %}
from typing import TYPE_CHECKING

from werkzeug.security import generate_password_hash

if TYPE_CHECKING:
    from flask import Flask


def init_commands(app: "Flask"):
    @app.cli.command("db_init")
    def db_init():
        """Create database and load initial data into database"""
        from .app import db
        from .models import User
        db.create_all()
        db.session.bulk_insert_mappings(
            User,
            [
                dict(
                    id=1,
                    email="your@email.address",
                    first_name="Your",
                    last_name="Name",
                    password=generate_password_hash("kodblok123")
                )
            ]
        )
        db.session.commit()
{% endhighlight %}

/instance/config.py
{% highlight python %}
import os

instance_path = os.environ.get(
    "FLASK_INSTANCE_PATH", os.path.join(os.getcwd(), "instance")
)

SECRET_KEY = b'h\xb0u\x41}%\x81Z\xec\x23\x1e\xa1\xc4q\x9an?\x83\x93\x94\x94\xc9Y:'

SESSION_TYPE = "filesystem"
SESSION_FILE_DIR = os.path.join(instance_path, "..", "data", "sessions")

# SESSION_TYPE = "memcached"
# SESSION_KEY_PREFIX = "yourproject:session:"

CSRF_COOKIE_TIMEOUT = None

SQLALCHEMY_DATABASE_URI = "sqlite:///../data/yourproject.sqlite"
# SQLALCHEMY_ENGINE_OPTIONS= {"options": "-c timezone=utc"}
SQLALCHEMY_TRACK_MODIFICATIONS = False


MAIL_SERVER = "localhost"
MAIL_PORT = "1025"
MAIL_DEFAULT_SENDER = "info@blokkod.me"

JEMBE_MEDIA_FOLDER = os.path.join("..", "data", "media")


# your project specific settings
APP_ROOT_URL="http://localhost:5000/"

{% endhighlight %}

/tests/conftest.py
{% highlight python %}
import pytest
from your_project import create_app


@pytest.fixture
def app():
    app = create_app({"TESTING": True})
    yield app

@pytest.fixture
def client(app):
    yield app.test_client()

{% endhighlight %}


Now you are ready to install Flask in your virtual environment and start developing

{% highlight shell-session %}
(.venv) $ pip install --upgrade pip
(.venv) $ pip install -e .[dev]
(.venv) $ flask run
{% endhighlight %}


And you are ready to go with development.

You want to start even faster, you don't want to copy paste code and manualy edit files above then:

{% highlight shell-session %}
(.venv) $ git clone github.com/peranp/start-flask-project
(.venv) $ ./configure_me.sh
(.venv) $ rm configure_me.sh
(.venv) $ flask run
{% endhighlight %}

All best and hapy development.
