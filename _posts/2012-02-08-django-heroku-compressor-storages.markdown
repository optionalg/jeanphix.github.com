---
layout: post
lang: en
title: Django, heroku & S3 FTW
tags:
- django
- webperf
- heroku
- S3
---

<p class="alert"><i>Outdated</i> Heroku now "automatically runs <a href="https://devcenter.heroku.com/articles/django-assets">collectstatic</a> on deployment.
    By the way, I encourage you to put all your environment specific configs (like database dsn, debug...) into environment variables.</p>

Today I'll show you my tips backed from a django project deployment over heroku (application) and S3 (statics).

## Setting up environments

First of all, I need two remote environments (production and staging) with distinct settings. In order to feet my git workflow, I had to create three local branches:

* master: that I push even on heroku _production_ application and on a _master_ github branch.
* staging: that I push even on heroku _staging_ application and on a _staging_ github branch.
* development: that I push on a _development_ github branch.

Project bootstrap:

{% highlight bash %}
$ pip install django gunicorn psycopg2
$ pip freeze > requirements.txt
$ git init
$ django-admin.py startproject myproject
$ python myproject/manage.py startapp myapp
{% endhighlight %}

Then in `myproject/settings.py` I add 'gunicorn' to INSTALLED_APPS.

{% highlight bash %}
$ echo 'web: python myproject/manage.py run_gunicorn -b "0.0.0.0:$PORT" -w 3' > Procfile
$ git add .
$ git commit -m 'Initial commit'
{% endhighlight %}

Production:

{% highlight bash %}
$ heroku create --stack cedar --remote production
$ git push production master
{% endhighlight %}

Staging:

{% highlight bash %}
$ git checkout -b staging
$ heroku create --stack cedar --remote staging
$ git push staging master
{% endhighlight %}

{% highlight bash %}
$ git checkout -b development
{% endhighlight %}

Then the workflow is as easy as:

* commit features' stuff on _development_
* merge _development_ into _staging_
* push _staging_ into remote _staging/master_
* merge _staging_ into _master_
* push _master_ to remote _production/master_

Now I need a way to setup environment specific settings, here is the pattern I used:

myproject/settings.py:

{% highlight python %}
import os
import imp
import socket
import subprocess


LOCAL_HOSTNAMES= ('myhost',)
HOSTNAME = socket.gethostname()

PROJECT_ROOT = os.path.dirname(os.path.realpath(__file__))

def get_environment_file_path(env):
    return os.path.join(PROJECT_ROOT, 'config', '%s.py' % env)

if 'APP_ENV' in os.environ:
    ENV = os.environ['APP_ENV']
elif HOSTNAME in LOCAL_HOSTNAMES:
    branch = subprocess.check_output(
        ['git', 'rev-parse', '--abbrev-ref', 'HEAD']).strip('\n')
    if os.path.isfile(get_environment_file_path(branch)):
        ENV = branch
    else:
        ENV = 'development'

try:
    config = imp.load_source('env_settings', get_environment_file_path(ENV))
    from env_settings import *
except IOError:
    exit("No configuration file found for env '%s'" % ENV)
{% endhighlight %}

Then I can put my common settings into settings.py and specific settings into config/{branch}.py

To get it work on remote env, just set the APP_ENV this way:

{% highlight bash %}
$ heroku config:add APP_ENV=staging --remote staging
$ heroku config:add APP_ENV=production --remote production
{% endhighlight %}

That's it, the application now switches to the appropriate config file for current branch and can be override by setting APP_ENV environment variable.

## django-compressor feat. django-storages

The other tip I'll show you here is to configure [django-compressor](https://github.com/jezdez/django_compressor) and [django-storages](http://code.larlet.fr/django-storages/) to work together on previous set environments.

The expected behaviours I need:

	* When settings.DEBUG is True:
	** statics have to be delivered by the application (or heroku nginx on remotes).
	** compressor has to be disabled
	* When settings.DEBUG is FALSE:
	** statics have to be delivered by Amazon S3 CDN.
	** compressor has to be enabled and upload compressed assets to S3

As compressor use "locally collected" statics, I need to collect it locally on remotes to, lets create a custom storage class, myproject/myapp/storage.py:

{% highlight python %}
from django.core.files.storage import get_storage_class
from storages.backends.s3boto import S3BotoStorage

class CachedS3BotoStorage(S3BotoStorage):
    """S3 storage backend that saves the files locally, too.
    """
    def __init__(self, *args, **kwargs):
        super(CachedS3BotoStorage, self).__init__(*args, **kwargs)
        self.local_storage = get_storage_class(
            "compressor.storage.CompressorFileStorage")()

    def save(self, name, content):
        name = super(CachedS3BotoStorage, self).save(name, content)
        self.local_storage._save(name, content)
        return name
{% endhighlight %}

Then I add settings this way:

{% highlight python %}
LOCAL_HOSTNAMES= ('myhost',)
# Statics
STATIC_URL = '/static/'

STATIC_ROOT = os.path.join(PROJECT_ROOT, 'static')

MEDIA_ROOT = os.path.join(STATIC_ROOT, 'media')
MEDIA_UPLOAD_ROOT = os.path.join(MEDIA_ROOT, 'uploads')

MEDIA_URL = STATIC_URL + 'media'

# Compressor
COMPRESS_ENABLED = DEBUG is False
if COMPRESS_ENABLED:
    COMPRESS_CSS_FILTERS = [
        'compressor.filters.css_default.CssAbsoluteFilter',
        'compressor.filters.cssmin.CSSMinFilter',
    ]
    COMPRESS_STORAGE = 'myapp.storage.CachedS3BotoStorage'
    COMPRESS_URL = STATIC_URL
    COMPRESS_OFFLINE = True

# Storages
if not DEBUG and HOSTNAME in LOCAL_HOSTNAMES:
    STATICFILES_STORAGE = 'myapp.storage.CachedS3BotoStorage'
{% endhighlight %}

Then add this lines on myproject/config/development.py:

{% highlight python %}
if not DEBUG:
    STATIC_URL = 'https://xxx-dev.s3.amazonaws.com/'

AWS_ACCESS_KEY_ID = "xxxxxxxxxxxxx"
AWS_SECRET_ACCESS_KEY = "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
AWS_STORAGE_BUCKET_NAME = "xxx-dev"
{% endhighlight %}

And modify the Profile this way:

{% highlight bash %}
web: python project/manage.py collectstatic --noinput; python myproject/manage.py compress; python myproject/manage.py run_gunicorn -b "0.0.0.0:$PORT" -w 3
{% endhighlight %}

Ok, so now, to collect static over S3 I just need to set myproject.config.{branch}.DEBUG to False and then run:

{% highlight bash %}
$ python myproject/manage.py collectstatic --noinput
{% endhighlight %}

_Note_: collecting statics via a _worker_ is probably a better practice but it will increase your heroku bills...

Hope this will help you.

See you.
