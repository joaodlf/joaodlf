---
title: "Getting started with Celery"
date: 2019-05-31
draft: true
keywords:
- gohugo
- hugo
- netlify
---

[Celery](http://www.celeryproject.org/) is a Distributed Task Queue written in Python. Task Queues are very useful for web/backend development, especially useful when dealing with time consuming operations that your web requests shouldn't be doing, ex:

- A new user registers in your site, now you need to send out an email, right? Don't send the email as part of the request, chuck it in the queue!
- You have a REST API that ingests data, maybe it needs to do some database checks? read/write operations? Maybe you don't need to return any of this data back? Don't do these checks as part of the request, chuck it in the queue!

The idea here is that you want to speed up requests for your users. You should never do anything that some other process can pick up on the background. To an extent, this is also good for your code: [DOTADIW](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well). Your request handlers should do one thing, and do it well. All extra functionality that has no impact on the actual response should be off loaded somewhere else.

### Setup

Instalation is performed via pip:

```bash
pip install celery
```

You will also need to pick a broker (used to send and receive messages), you have two major choices: **RabbitMQ** and **Redis**. My recommendation for any serious project: **Use RabbitMQ**. Redis is dead simple to install and use, and I already have some experience with it, so when I first started out with Celery it just felt natural to use Redis for my broker. Unfortunately, as my data ingestion grew, I started getting a few [issues](https://github.com/andymccurdy/redis-py/issues/968). For this reason, I decided to use RabbitMQ.

Once RabbitMQ is installed and running, you'll need to setup a celery user within RabbitMQ:

```bash
rabbitmqctl add_user celery <password>
rabbitmqctl add_vhost celery
rabbitmqctl set_user_tags celery administrator
rabbitmqctl set_permissions -p celery celery ".*" ".*" ".*"
```

You are now ready to start using Celery in your python code:

```python
# Using RabbitMQ.
celery = Celery(__name__, broker="amqp://celery:<password>@<host>host:5672/celery")
# Using Redis.
celery = Celery(__name__, broker="redis://:<password>@<host>:6379/0")
```

The Redis example above assumes you have setup a password. I find Redis a suitable and easy to use broker for development environments.

### Workers

You will need at least one celery worker running to be able to execute tasks:

```bash
celery worker -A run.celery -l DEBUG
```

A note on the flags:

- `-A` is where you tell the Celery worker where to find your application.
- `-l` is the log level.

Celery works in a distributed fashion, which means you can have workers running on multiple machines - ideal as your ingestion rate grows.

### Tasks

Celery is all about the concept of tasks, which are just functions wrapped with celery's task decorator:

```python
@celery.task()
def send_welcome_email(email: str):
    # Logic to send an email here...
```

#### Delay

`.delay()` is the straightforward way to send a task message:

```python
send_welcome_email.delay("joao@example.com")
```

This will send this task to the broker, which in turn will be processed by one of the workers. Execution will continue past `.delay()` in an asynchronous fashion.

#### Autoretry

Following our example above, lets assume we need to send the email to an ESP (such as Mailchimp or SendGrid), these services will typically offer an API we can use in order to communicate with the service. APIs can, and will, fail. Celery allows us to retry our tasks in case of failures, we could modify our task like so:

```python
@celery.task(bind=True)
def send_welcome_email(self, email):
    response = requests.post("broken endpoint API", params={"email": email})

    try:
        response.raise_for_status()
    except requests.exceptions.HTTPError as e:
        raise self.retry(exc=e)
```

Forcing a retry can be helpful in some cases, but in this case we already know what exception to expect, for this reason we can use `autoretry_for`:

```python
@celery.task(autoretry_for=(requests.exceptions.HTTPError,))
def send_welcome_email(email):
    response = requests.post("broken endpoint API", params={"email": email})
    response.raise_for_status()
```

We can take this a step further by introducing an [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) via the `retry_backoff` option:

```python
@celery.task(autoretry_for=(requests.exceptions.HTTPError,), retry_backoff=True)
def send_welcome_email(email):
    response = requests.post("broken endpoint API", params={"email": email})
    response.raise_for_status()
```

Might not fit all use cases, but still very handy to have it available by default!
