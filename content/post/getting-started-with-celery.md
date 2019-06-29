---
title: "Getting started with Celery"
date: 2019-06-29
keywords:
- celery
- python
- flask
---

[Celery](http://www.celeryproject.org/) is a Distributed Task Queue written in Python. Task Queues are very useful for web/backend development, especially when dealing with time consuming operations that your web requests shouldn't be doing, ex:

- A new user registers to your site, now you need to send them an email, right? Don't send the email as part of the request, chuck it in the queue!
- You have a REST API that ingests data, maybe it needs to do some database checks? read/write operations? Maybe you don't need to return any of this data back? Don't do these operations as part of the request, chuck them in the queue!

The idea here is that you want to speed up requests for your users. You should never do anything that some other process can pick up on the background. To an extent, this is also good for your code: [DOTADIW](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well). Your request handlers should do one thing, and do it well. All extra functionality that has no impact on the actual response should be off loaded somewhere else.

### Why Celery?

Celery has been around for a [long time](https://docs.celeryproject.org/en/latest/history/changelog-1.0.html) now. It's well documented, battle tested and easy to scale up. Sure, there are other, trendier, big projects out there, but if you are already running a Python stack, Celery can just slot right in.

In my case, most of my Python web development is Flask based, which Celery easily integrates with (more on this later...).

### Setup

Installation is performed via pip:

```bash
pip install celery
```

You will also need to pick a broker (used to send and receive messages), you have two major choices: **RabbitMQ** and **Redis**. My recommendation for any serious project: **Use RabbitMQ**. Redis is dead simple to install and use, and I already have some experience with it, so when I first started out with Celery it just felt natural to use Redis for my broker. Unfortunately, as my data ingestion grew, I started getting a few [issues](https://github.com/andymccurdy/redis-py/issues/968). For this reason, I decided to use RabbitMQ - installation instructions [here](https://www.rabbitmq.com/download.html).

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

You will need at least one Celery worker running to be able to execute tasks:

```bash
celery worker -A run.celery -l DEBUG
```

A note on the flags:

- `-A` is where to find the Celery application. In this example, you have a module called `run`, and a `celery` application within this module.
- `-l` is the log level.

Celery works in a distributed fashion, which means you can have workers running on multiple machines - ideal as your ingestion rate grows and you need to scale up.

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

Following our example above, lets assume we need to trigger an email from an ESP (such as Mailchimp or SendGrid), these services will typically offer an HTTP API we can use in order to communicate with the service. APIs can, and will, fail. Celery allows us to retry our tasks in case of failures. We could modify our task like so:

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

Might not fit all use cases, but hopefully this demonstrates how feature rich Celery is!

### Monitoring

Our tasks are running smoothly, but how can we keep an eye on performance, results and general monitoring? Fortunately, Celery offers plenty of [options](https://docs.celeryproject.org/en/latest/userguide/monitoring.html). One that is definitely worth mentioning is [Flower](https://flower.readthedocs.io/en/latest/), a web based GUI.

Installation is one more time performed via pip:

```bash
pip install flower
```

Starting Flower is also straightforward:

```bash
flower -A run.celery --port=5555
```

If running locally, you should now be able to navigate to http://localhost:5555/ and have a look around. Flower offers plenty of insight:

- Worker list, including status and various figures related to task execution.
- Task list, including a search functionality, filtering, etc.
- Insight into individual tasks, including execution times, returned results, etc.
- Broker information.
- Real time performance graphs.

### Integration with existing code

As previously mentioned, Celery can easily integrate with most Python projects. I'll finish off this post with a Flask example:

```python
import requests
from celery import Celery
from flask import Flask

app = Flask(__name__)
celery = Celery(__name__, broker="redis://:<password>@<host>:6379/0")


@app.route("/sign-up")
def signup():
    # Sign Up logic here...

    # ... And eventually trigger a task.
    send_welcome_email.delay(email)
    return "Signed Up!"


@celery.task(autoretry_for=(requests.exceptions.HTTPError,), retry_backoff=True)
def send_welcome_email(email):
    response = requests.post("ESP API endpoint...", params={"email": email})
    response.raise_for_status()
    return email
```

Hopefully this demonstrates how easy it is to accommodate Celery into an existing project. As you can see, there is no magic involved or need for bridging libraries.
