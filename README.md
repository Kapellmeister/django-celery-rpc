django-celery-rpc
=================

[![Build Status](https://travis-ci.org/ttyS15/django-celery-rpc.svg)](https://travis-ci.org/ttyS15/django-celery-rpc)

Remote access from one system to models and functions of other one using Celery machinery.

Relies on three outstanding python projects:

 - [Celery](http://www.celeryproject.org/)
 - [Django Rest Framework](http://www.djang)
 - [Django](https://www.djangoproject.com/)

## Main features

Client and server are designed to:

 - filter models with Django ORM lookups, Q-objects and excludes;
 - change model state (create, update, update or create, delete);
 - change model state in bulk mode (more than one object per request);
 - atomic get-set model state with bulk mode support;
 - call function;
 - client does not require Django;

## Basic Configuration

Default configuration of **django-celery-rpc** must be overridden in settings.py by **CELERY_RPC_CONFIG**.
The **CELERY_RPC_CONFIG** is a dict which must contains at least two keys: **BROKER_URL** and **CELERY_RESULT_BACKEND**.
Any Celery config params also permitted
(see [Configuration and defaults](http://celery.readthedocs.org/en/latest/configuration.html))

### server **span**

setting.py:

```python
# minimal required configuration
CELERY_RPC_CONFIG = {
	'BROKER_URL': amqp://10.9.200.1/,
	'CELERY_RESULT_BACKEND': 'redis://10.9.200.2/0',
}
```

### server **eggs**

setting.py:

```python
# alternate request queue and routing key
CELERY_RPC_CONFIG = {
	'BROKER_URL': amqp://10.9.200.1/,
	'CELERY_RESULT_BACKEND': amqp://10.9.200.1/',
	'CELERY_DEFAULT_QUEUE': 'celery_rpc.requests.alter_queue',
	'CELERY_DEFAULT_ROUTING_KEY': 'celery_rpc.alter_routing_key'
}
```

### client

setting.py:

```python
# this settings will be used in clients by default
CELERY_RPC_CONFIG = {
	'BROKER_URL': amqp://10.9.200.1/,
	'CELERY_RESULT_BACKEND': 'redis://10.9.200.2/0',
}

# 'eggs' alternative configuration will be explicitly passed to the client constructor
CELERY_RPC_EGGS_CLIENT = {
	# BROKER_URL will be used by default from section above
	'CELERY_RESULT_BACKEND': amqp://10.9.200.1/',
	'CELERY_DEFAULT_QUEUE': 'celery_rpc.requests.alter_queue',
	'CELERY_DEFAULT_ROUTING_KEY': 'celery_rpc.alter_routing_key'
}
```

*Note:
1. client and server must share the same __BROKER_URL__, __RESULT_BACKEND__, __DEFAULT_EXCHANGE__, __DEFAULT_QUEUE__, __DEFAULT_ROUTING_KEY__
2. different server must serve different request queues with different routing keys or must work with different exchanges*

example.py

```python
from celery_rpc.client import Client
from django.conf import settings

# create client with default settings
span_client = Client()

# create client for `eggs` server
eggs_client = Client(CELERY_RPC_EGGS_CLIENT)
```

## Using client

You can find more examples in tests.

### Filtering

Simple filtering example

```
span_client.filter('app.models:MyModel', kwargs=dict(filter={'a__exact':'a'}))
```

Filtering with Q object

```
from django.db.models import Q
span_client.filter('app.models:MyModel', kwargs=dict(filters_Q=(Q(a='1') | Q(b='1')))
```

Also, we can use both Q and lookups

```
span_client.filter('app.models:MyModel', kwargs=dict(filters={'c__exact':'c'}, filters_Q=(Q(a='1') | Q(b='1')))
```

Exclude supported

```
span_client.filter('app.models:MyModel', kwargs=dict(exclude={'c__exact':'c'}, exclude_Q=(Q(a='1') | Q(b='1')))
```

You can mix filters and exclude, Q-object with lookups. Try it yourself. ;)

Full list of available kwargs:

    filters - dict of terms compatible with django lookup fields
    offset - offset from which return a results
    limit - max number of results
    fields - list of serializer fields, which will be returned
    exclude - lookups for excluding matched models
    order_by - order of results (list, tuple or string),
        minus ('-') set reverse order, default = []
    filters_Q - django Q-object for filtering models
    exclude_Q - django Q-object for excluding matched models


List of all MyModel objects with high priority

```
span_client.filter('app.models:MyModel', high_priority=True)
```

### Creating

Create one object

```
span_client.create('apps.models:MyModel', data={"a": "a"})
```

Bulk creating

```
span_client.create('apps.models:MyModel', data=[{"a": "a"}, {"a": "b"}])
```

### Pipe

It's possible to pipeline tasks, so they will be executed in one transaction.

```python
p = span_client.pipe()
p = p.create('apps.models:MyModel', data={"a": "a"})
p = p.create('apps.models:MyAnotherModel', data={"b": "b"})
p.run()
```

You can pass some arguments from previous task to the next.

Suppose you have those models on the server

```python
class MyModel(models.Model):
    a = models.CharField()
    
class MyAnotherModel(models.Model):
    fk = models.ForeignKey(MyModel)
    b = models.CharField()
```

You need to create instance of MyModel and instance of MyAnotherModel which reffers to MyModel

```python
p = span_client.pipe()
p = p.create('apps.models:MyModel', data={"a": "a"})
p = p.transform({"fk": "id"}, defaults={"b": "b"})
p = p.create('apps.models:MyAnotherModel')
p.run()
```

In this example the `transform` task: 
 - take result of the previous `create` task
 - extract value of "id" field from it
 - add this value to "defaults" by key "fk"
 
After that next `create` task takes result of `transform` as input data

### Add/delete m2m relations

Lets take such models:

```python
class MyModel(models.Model):
    str = models.CharField()
    
class MyManyToManyModel(models.Model):
    m2m = models.ManyToManyField(MyModel, null=True)
```

Add relation between existing objects

```python
my_models = span_client.create('apps.models:MyModel' [{'str': 'monthy'}, {'str': 'python'}])
m2m_model = span_client.create('apps.models:MyManyToManyModel', {m2m: [my_models[0]['id']]})

# Will add 'python' to m2m_model.m2m where 'monty' already is
data = dict('mymodel': my_models[1]['id'], 'mymanytomanymodel': m2m_model['id'])
through = span_client.create('apps.models:MyManyToManyModel.m2m.through', data)
```

And then delete some of existing relations

```python
# Next `pipe` will eliminate all relations where `mymodel__str` equals 'monty'
p = span_client.pipe()
p = p.filter('apps.models:MyManyToManyModel.m2m.through', {'mymodel__str': 'monthy'})
p = p.delete('apps.models:MyManyToManyModel.m2m.through')
p.run()
```

## Run server instance

```python
celery worker -A celery_rpc.app
```

Server with support task consuming prioritization

```python
celery multi start 2 -A celery_rpc.app -Q:1 celery_rpc.requests.high_priority
```

*Note, you must replace 'celery_rpc.request' with actual value of config param __CELERY_DEFAULT_QUEUE__*

Command will start two instances. First instance will consume from high priority queue only. Second instance will serve both queues.

For daemonization see [Running the worker as a daemon](http://celery.readthedocs.org/en/latest/tutorials/daemonizing.html)

## Run tests

```shell
python django-celery-rpc/celery_rpc/runtests/runtests.py
```

## More Configuration

### Overriding base task class

```python
OVERRIDE_BASE_TASKS = {
    'ModelTask': 'package.module.MyModelTask',
    'ModelChangeTask': 'package.module.MyModelChangeTask',
    'FunctionTask': 'package.module.MyFunctionTask'
}

Supported class names: ModelTask, ModelChangeTask, FunctionTask

```

## TODO

 - Set default non-generic model serializer.
 - Test support for RPC result backend from Celery.
 - Token auth and permissions support (like DRF).
 - Resource map and strict mode.
 - ...
 
## Acknowledgements

Thanks for all who contributing to this project:
 - https://github.com/tumbler
 - https://github.com/voron3x
 - https://github.com/dtrst
 - https://github.com/anatoliy-larin
 - https://github.com/bourivouh
 - https://github.com/tumb1er
