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

### Usage

I'm going to be using Flask to exemplify common Celery use cases.

#### Delay

