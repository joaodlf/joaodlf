---
title: "Python for the Web"
date: 2018-02-18
keywords:
- flask
- python
- peewee python
- uwsgi
---

Having worked on so many Python Web projects recently, it became obvious I needed some kind of boilerplate to get new projects
started, hence [flask-boilerplate](https://github.com/joaodlf/flask-boilerplate).

A lof of this is based on the excellent [cookiecutter-flask](https://github.com/sloria/cookiecutter-flask) -
I've learned a lot from it, but made enough modifications and additions to call this my own.

I have a few requirements, the most important one being **simplicity**. I dislike large dependencies and
complicated "pythonic" code. Typically, I will also need some sort of CLI (for crons, scripts, etc),
so I try to fulfil that requirement, on top of the web stack.

Let's go over some of the requirements,

### Flask 

[Flask](https://github.com/pallets/flask) is simple and comes with all essentials. 
The routing system is accessible, and concepts like blueprints tie it all up into a nice package. Programming languages
aside, I almost always favour a small framework like Flask. Other, large and opinionated frameworks (like Django)
don't click with me. This isn't to say I would never use a large framework, I believe that it all depends on the
scope and size of the project (as well as the development team).

Very importantly, **Flask allows me to develop around it, with ease.**

### uWSGI

[uWSGI](https://github.com/unbit/uwsgi) is my preferred way to serve web applications. I like to
run it in both development and production, with different configuration files. This forces me and other developers to
be in contact with uWSGI all the time, and not just in a production environment.

To run the webserver locally, simply execute `uwsgi --ini web/uwsgi/development.ini` from the command line.

The `development.ini` file includes all the required settings for a smooth development experience:

```config
[uwsgi]
http = 127.0.0.1:80
home = env
wsgi-file = web_server.py
callable = app
py-autoreload = 1
master = true
reload-mercy = 0
worker-reload-mercy = 0
enable-threads = true
touch-reload = web/uwsgi/development.ini
```

`reload-mercy` and `worker-reload-mercy` will take care of ensuring fast reload times. `py-autoreload` will detect changes
to any .py file and trigger an automatic reload.

For production, you'll want to introduce a few more settings, such as `processes` and `threads` -
More details on the official uWSGI [docs](http://uwsgi-docs.readthedocs.io/en/latest/Configuration.html).

In the past, we have also used Apache + [mod_wsgi](http://modwsgi.readthedocs.io/en/develop/) for
production, but that presented a few issues that I would rather not get into in this post.

(**Note:** In production, I also run [NGINX](https://www.nginx.com/) on top of uWSGI)

### Peewee

Unpopular opinion: I dislike [SQLAlchemy](https://www.sqlalchemy.org/). It's the opposite of simplicity.
Yes, even when just using "Core".

In another life, I did quite a bit of PHP programming, and for all its flaws, the PHP community did
one thing right: SQL. A vast amount of ORMs and query builders gave developers a lot of choice. This is not the case
with Python. The community is happy with SQLAlchemy, and that's it. Very little development has gone towards
alternatives - A curious case in the vast world of choice in Python. A good example is the utter lack of actively
supported SQL query builders (which I tend to prefer over full blown ORMs).

Not all is doom and gloom, though! -  [peewee](https://github.com/coleifer/peewee) to the rescue!
A simple and expressive way to handle SQL. It's the only ORM I seem to tolerate in Python.

Here is a peewee model example from `/models/api_ip_whitelist.py` (A Postgres table):

```python
import peewee
from peewee import *

from models import BaseModel


class ApiIpWhitelist(BaseModel):
    comment = CharField(null=True)
    dt_created = DateTimeField()
    ip_address = CharField()  # inet field, peewee does not support this type!

    class Meta:
        db_table = "api_ip_whitelist"

    @staticmethod
    def is_valid_ip(ip_address: str):
        """
        Validates a single IP (i.e. is it in the database?).
        """
        # Check ip.
        try:
            valid = ApiIpWhitelist.select().where(ApiIpWhitelist.ip_address == ip_address).get()
        except ApiIpWhitelist.DoesNotExist:
            valid = False

        if not valid:
            # Check ip range.
            try:
                valid = ApiIpWhitelist.raw(
                    "SELECT * FROM api_ip_whitelist WHERE ip_address >> %s::inet;", ip_address
                ).execute()
            except peewee.DataError:
                valid = False

        if valid:
            return True

        return False
```

For integration with Flask, there is [flask-peewee](https://github.com/coleifer/flask-peewee). It works
great, but since I want to take advantage of peewee in both a web and cli environment, I tend to skip it.

(**Note:** Postgres is my choice for a RDBMS, which peewee supports handsomely via the
[playhouse extension](http://docs.peewee-orm.com/en/latest/peewee/playhouse.html#postgres-ext))

### Fire

[Python Fire](https://github.com/google/python-fire) by Google is the answer for my CLIs.

A CLI has become essential in my web projects, typically in the form of crons or small management scripts. This is why
I like to decouple my dependencies from the web aspect of the project.

A simple CLI example is provided in `cli/example.py`:

```python
import fire
from logzero import logger

from cli import Cli


class Example(Cli):
    def __init__(self):
        Cli.__init__(self)

    def example(self, num1: int = 2, num2: int = 2):
        self.start("example")

        total = num1 + num2
        total_minus_one = total - 1

        logger.info(f"{num1} plus {num2} is {total}")
        logger.info(f"minus 1 that's {total_minus_one}")
        logger.warn("QUICK MATHS")


if __name__ == "__main__":
    fire.Fire(Example)
```

Simply run `python cli/example.py example` from your command line to execute.

(**Note:** [Click](http://click.pocoo.org/5/) is also a good alternative that I have used in the past.)

### Logzero

Logging in Python via the [logging](https://docs.python.org/3/library/logging.html) is fantastic,
but I always end up with a lot of boilerplate code to get it to where I want it to be - Logging to files, formatting,
levels, colors, file rotation/removal... It's a lot. One day I found [Logzero](https://github.com/metachris/logzero)
and never went back. The defaults are sensible and it has all the features I need.

Here is the very simple setup from the `cli/__init__.py` file:

```python
# Set the logger.
logzero.loglevel(logging.INFO)
logzero.logfile(f"{self.logs_dir}/{self.name}.log", maxBytes=1000000, backupCount=3)
```

### Sentry (optional)

[Sentry](https://sentry.io/)  styles itself as "Error tracking software".
It has become a staple of my projects (not just Python!). I've made it an optional in
[flask-boilerplate](https://github.com/joaodlf/flask-boilerplate), but I definitely recommend it.
There is a free plan for small projects, and you can even [host it](https://docs.sentry.io/server/) yourself!