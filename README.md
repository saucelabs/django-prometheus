# django-prometheus
Export Django monitoring metrics for Prometheus.io

[![PyPI version](https://badge.fury.io/py/django-prometheus.svg)](http://badge.fury.io/py/django-prometheus)
[![Build Status](https://travis-ci.org/korfuri/django-prometheus.svg?branch=master)](https://travis-ci.org/korfuri/django-prometheus)
[![PyPi page link -- Python versions](https://img.shields.io/pypi/pyversions/django-prometheus.svg)](https://pypi.python.org/pypi/django-prometheus)


## Usage

### Requirements

* Django >= 1.8

### Installation

Install with:
```shell
pip install django-prometheus
```

Or, if you're using a development version cloned from this repository:
```shell
python path-to-where-you-cloned-django-prometheus/setup.py install
```

This will install [prometheus_client](https://github.com/prometheus/client_python) as a dependency.

### Quickstart

In your settings.py:

```python
INSTALLED_APPS = (
   ...
   'django_prometheus',
   ...
)

MIDDLEWARE_CLASSES = (
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # All your other middlewares go here, including the default
    # middlewares like SessionMiddleware, CommonMiddleware,
    # CsrfViewmiddleware, SecurityMiddleware, etc.
    'django_prometheus.middleware.PrometheusAfterMiddleware',
)
```

In your urls.py:

```python
urlpatterns = [
    ...
    url('', include('django_prometheus.urls')),
]
```

### Monitoring your databases

SQLite, MySQL, and PostgreSQL databases can be monitored. Just
replace the `ENGINE` property of your database, replacing
`django.db.backends` with `django_prometheus.db.backends`.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django_prometheus.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    },
}
```

### Monitoring your caches

Filebased, memcached, redis caches can be monitored. Just replace
the cache backend to use the one provided by django_prometheus
`django.core.cache.backends` with `django_prometheus.cache.backends`.

```python
CACHES = {
    'default': {
        'BACKEND': 'django_prometheus.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```

### Monitoring your models

You may want to monitor the creation/deletion/update rate for your
model. This can be done by adding a mixin to them. This is safe to do
on existing models (it does not require a migration).

If your model is:

```python
class Dog(models.Model):
    name = models.CharField(max_length=100, unique=True)
    breed = models.CharField(max_length=100, blank=True, null=True)
    age = models.PositiveIntegerField(blank=True, null=True)
```

Just add the `ExportModelOperationsMixin` as such:

```python
from django_prometheus.models import ExportModelOperationsMixin

class Dog(ExportModelOperationsMixin('dog'), models.Model):
    name = models.CharField(max_length=100, unique=True)
    breed = models.CharField(max_length=100, blank=True, null=True)
    age = models.PositiveIntegerField(blank=True, null=True)
```

This will export 3 metrics, `django_model_inserts_total{model="dog"}`,
`django_model_updates_total{model="dog"}` and
`django_model_deletes_total{model="dog"}`.

Note that the exported metrics are counters of creations,
modifications and deletions done in the current process. They are not
gauges of the number of objects in the model.

Starting with Django 1.7, migrations are also monitored. Two gauges
are exported, `django_migrations_applied_by_connection` and
`django_migrations_unapplied_by_connection`. You may want to alert if
there are unapplied migrations.

If you want to disable the Django migration metrics, set the
`PROMETHEUS_EXPORT_MIGRATIONS` setting to False.


### Monitoring and aggregating the metrics

Prometheus is quite easy to set up. An example prometheus.conf to
scrape `127.0.0.1:8001` can be found in `examples/prometheus`.

Here's an example of a PromDash displaying some of the metrics
collected by django-prometheus:

![Example dashboard](https://raw.githubusercontent.com/korfuri/django-prometheus/master/examples/django-promdash.png)

## Adding your own metrics

You can add application-level metrics in your code by using
[prometheus_client](https://github.com/prometheus/client_python)
directly. The exporter is global and will pick up your metrics.

To add metrics to the Django internals, the easiest way is to extend
django-prometheus' classes. Please consider contributing your metrics,
pull requests are welcome. Make sure to read the Prometheus best
practices on
[instrumentation](http://prometheus.io/docs/practices/instrumentation/)
and [naming](http://prometheus.io/docs/practices/naming/).

## Importing Django Prometheus using only local settings

If you wish to use Django Prometheus but are not able to change
the code base, it's possible to have all the default metrics by
modifying only the settings.

First step is to inject prometheus' middlewares and to add
django_prometheus in INSTALLED_APPS

```python
MIDDLEWARE = (
        ('django_prometheus.middleware.PrometheusBeforeMiddleware',) +
       MIDDLEWARE +
        ('django_prometheus.middleware.PrometheusAfterMiddleware',)
    )

INSTALLED_APPS = INSTALLED_APPS + ('django_prometheus',)
```

Second step is to create the /metrics end point, for that we need
another file (called urls_prometheus_wrapper.py in this example) that
will wraps the apps URLs and add one on top:


```python
from django.conf.urls import include, url


urlpatterns = []

urlpatterns.append(url('^prometheus/', include('django_prometheus.urls')))
urlpatterns.append(url('', include('myapp.urls')))
```

This file will add a "/prometheus/metrics" end point to the URLs of django
that will export the metrics (replace myapp by your project name).

Then we inject the wrapper in settings:

```python 
ROOT_URLCONF = "graphite.urls_prometheus_wrapper"
```
