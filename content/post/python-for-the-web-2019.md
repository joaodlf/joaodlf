---
title: "Python for the Web (in 2019)"
date: 2019-02-09
keywords:
- flask
- python
- peewee python
- uwsgi
---

Last year I wrote about [Python for the Web](/python-for-the-web.html) - a boilerplate project to
kickoff my web development. It's been a year since I wrote that post, since then I have started a multitude of projects.
Along the way I have had the need need to update and upgrade the project structure, as well as some of the requirements.

The original [GitHub project](https://github.com/joaodlf/flask-boilerplate) has been updated
with these changes. So, what has changed?

### Project structure

A few things worth mentioning: The config file(s) now live in the `config` directory, this allows for easier config loading
via the `__init__.py` file:

```python
import sys

if "pytest" in sys.modules:
    from config.test_config import *
else:
    from config.config import *
```

As you can see, I am loading a different config when running under **pytest** - yes, the project now includes tests too,
which can be found in the `tests` dir. The tests are very basic and serve purely as examples to kickoff more serious
testing, but they will hopefully demonstrate how to bootstrap a test database:

```python
@with_test_db((ApiIpWhitelist,))
def test_general_ping_success():
    ApiIpWhitelist.create(
        ip_address="127.0.0.1"
    )

    response = testing_app.get("/api/general/ping")
    assert response.status_code == 200
```

 Notice the decorator wrapping the test function. It takes a tuple of models as an argument to create a new database for each
 individual test:

```python
def with_test_db(models: tuple):
    def decorator(func):
        @wraps(func)
        def test_db_closure(*args, **kwargs):
            test_db = PostgresqlDatabase(SQL_DATABASE, user=SQL_USER, password=SQL_PASSWORD, host=SQL_HOST)
            with test_db.bind_ctx(models):
                test_db.create_tables(models)
                try:
                    func(*args, **kwargs)
                finally:
                    test_db.drop_tables(models)

        return test_db_closure

    return decorator
```

This is a really nice decorator for testing with Flask and Peewee. I have to thank David Love (one of my colleagues)
for this one. You can read more about it in his own
[blog post](https://www.dvlv.co.uk/a-super-helpful-decorator-for-peeweeflask-unit-testing.html).

The `create_app` function in `web/app.py` now takes a `config_name` parameter. This makes it easier to load different Flask
configs depending on the environment: development, testing and production. The `create_app` function has also been modified
to follow an [_app factory_](http://flask.pocoo.org/docs/1.0/patterns/appfactories/).

Lastly, the `web/views` directory includes a `blueprint` file, where all the blueprints now reside:

```python
from flask import Blueprint

api_blueprint = Blueprint("api", __name__, url_prefix="/api")

public_blueprint = Blueprint("public", __name__)
```

This is purely for better code organization. I find that it helps having all the blueprints in the same file, above everything
else the views directory  - this avoids some ugly code down the line, such as imports at the end of files.

### Libraries

A lot of the old libraries are still being used: **Flask**, **uWSGi**, **Peewee**, **Logzero** and **Sentry**.

Most have been updated, though. **Flask** has finally reached version 1.0. **Peewee** has been updated to the latest
version and continues to grow as my prefered way to handle SQL in Python (huge shout-out to
[coleifer](https://github.com/coleifer)). **Sentry** has a flashy new API, a major improvement
over the last version.

**Fire** and **Arrow** have been replaced with **docopt** and **Pendulum**, respectfully.

#### docopt

I started running into a few issues with Fire, especially in regards
to [flag parsing](https://github.com/google/python-fire/issues/102).

I managed to get around these issues, but eventually stumbled upon docopt and immediately fell in love with it. It doesn't take long
to learn the so called *command-line interface description language*, it's very intuitive if you are used to well documented
CLI tools.

The [example CLI script](https://github.com/joaodlf/flask-boilerplate/blob/master/cli/example.py)
has been updated.

#### Pendulum

I didn't mention [Arrow](https://arrow.readthedocs.io/en/latest/) in my previous post, but at the time
it was my preferred date/time library.

I discovered [Pendulum](https://pendulum.eustace.io/) in 2018, and even though I don't necessarily have
much of a complaint about Arrow, I have to say the Pendulum API just feels a lot better, natural and intuitive. I've since
replaced Arrow in all of my codebases.

### Migrations

Migrations are something I have always disregarded, simply because I had never been able to find a solution that suited my needs.

I really dislike most migration tools that are out there in the wild. They are usually over complicated, tied to other
frameworks, and typically abstract SQL away - which I am really not a fan of.

In 2018 I was introduced to [Flyway](https://flywaydb.org/). It's exactly what I was looking for. All
it needs is a migrations directory and a basic config file. The migration files are pure SQL, flyway keeps track of what
files need to run by creating a table in your database (`flyway_schema_history`).

All you need to do is run `flyway migrate`, or if you want to implement it on an existing database, generate your initial schema
file and run `flyway baseline`.

Migrations via flyway can now be found under the `migrations` dir. Some more details on the flyway configuration are included in the
README.