---
layout: post
title: Heroku, Django, Nginx and PgBouncer
modified:
categories:
description: My struggle to get Heroku to work with Nginx and PgBouncer
tags: [heroku, django, nginx, pgbouncer, django q, uwsgi, gunicorn, waitress, python]
image:
  feature: sunset_2.jpg
  credit: koed00
  creditlink:
comments:
share: true
date: 2015-08-02T11:33:41+02:00
---

I've been struggling all weekend to get our [Heroku](https://www.heroku.com) servers set up with [Nginx](http://nginx.org/) and [PgBouncer](https://pgbouncer.github.io/). The simple part was finding out about Heroku buildpacks, with which you can add extra software to your dynos. The hard part, for me, was that most information on the Internet about using buildpacks seems to be outdated. Time to blog about it.

I'm going to assume you already have a basic understanding of running Django on Heroku and that you're using the new Cedar-14 stack.
First of all , let's introduce the players:

###Nginx

Nginx is a web server, but most widely used as a reverse proxy.
I'm using the [https://github.com/beanieboi/nginx-buildpack.git](https://github.com/beanieboi/nginx-buildpack.git) buildpack cause it uses the newer 1.8.0 stable version of nginx, but you can use the original [buildpack](https://github.com/ryandotsmith/nginx-buildpack) by @ryandotsmith.
Why do I use it on Heroku? I've been testing our applications for quite some by flooding them using tools like [Blitz](https://www.blitz.io/). We use [Gunicorn](http://gunicorn.org/), [uWSGI](http://uwsgi-docs.readthedocs.org/en/latest/) and [Waitress](http://waitress.readthedocs.org/en/latest/) as our wsgi servers and all of them fail if fed with enough long running queries and almost never recover. With Nginx guarding the gates, things will still shut down, but the webservers usually recover after the flood stops.

###PgBouncer
With Django resetting database connections on every request, you quickly run out of precious connections. Add to that a couple of async task queue workers and that connection count goes up fast. PgBouncer is a standalone connection pool for PostgreSQL. Your processes connect to PgBouncer and it, in turn, connects to your database server. Instead of breaking that connection, it's kept around for the next request. You're bound to see some performance improvements, since your project will spend less time setting up connections to your database.
You actually pay for the maximum connections your PostgreSQL server can handle on Heroku, so there is an economical aspect to using this buildpack asswell.

I used the [https://github.com/heroku/heroku-buildpack-pgbouncer](https://github.com/heroku/heroku-buildpack-pgbouncer) buildpack by @gregburek, although it's a slightly older version of pgbouncer, it is actively maintained and has the added benefit of using [stunnel](https://www.stunnel.org/index.html) to securely connect to your database over SSL.

###Heroku
My favorite PaaS. You will hear people complain about how they can do it cheaper and easier with their own stacks, but the fact is that it's been a time and money saver for us.
I spend near zero time on server maintenance, we've had no real downtime and security issues are always fixed within hours. We are using the new Cedar-14 stack, based on Ubuntu LTS 14.04.
Since the older stacks are deprecated everything in this article works under the assumption that you too are running Cedar-14.
You'll need the [Heroku Toolbelt](https://toolbelt.heroku.com/) to configure the buildpacks, so go ahead and install it if you haven't already.

### Setting up the buildpacks

Originally Heroku only supported a single buildpack for you dynos which had to be set through the `BUILDPACK_URL` configuration setting.
Then came @ddollar with his multipack, that allowed for extra buildpacks to be loaded through a `.buildpacks` file which you had to add to your project.
Heroku seems to have extended the `buildpacks` command so it's no longer neccesary to hack it like that:

{% highlight bash %}
Usage: heroku buildpacks

 display the buildpack_url(s) for an app

Examples:

 $ heroku buildpacks
 https://github.com/heroku/heroku-buildpack-ruby

Additional commands, type "heroku help COMMAND" for more details:

  buildpacks:add BUILDPACK_URL       #  add new app buildpack, inserting into list of buildpacks if neccessary
  buildpacks:clear                   #  clear all buildpacks set on the app
  buildpacks:remove [BUILDPACK_URL]  #  remove a buildpack set on the app
  buildpacks:set BUILDPACK_URL       #  set new app buildpack, overwriting into list of buildpacks if neccessary

{% endhighlight %}

Let's go ahead and add the Nginx buildpack. Make sure you are in your projects root directory when executing the `heroku` command, so it knows which application to target.
Otherwise append the `-a APPNAME` option after each command, to point it in the right direction.

{% highlight bash %}
$ heroku buildpacks:add https://github.com/beanieboi/nginx-buildpack.git
Buildpack added. Next release on myproject will use https://github.com/beanieboi/nginx-buildpack.git.
Run `git push heroku master` to create a new release using this buildpack.

{% endhighlight %}

Then the PgBouncer buildpack:

{% highlight bash %}
$ heroku buildpacks:add https://github.com/heroku/heroku-buildpack-pgbouncer.git
Buildpack added. Next release on myproject will use:
  1. https://github.com/beanieboi/nginx-buildpack.git
  2. https://github.com/heroku/heroku-buildpack-pgbouncer.git
Run `git push heroku master` to create a new release using these buildpacks.
{% endhighlight %}

That worked out well and is a lot easier than the old way of doing things.
Now we only have to add a Python buildpack. Normally Heroku does this automatically based on your project files, but we've
overridden this behavior with our custom buildpacks.

{% highlight bash %}
$ heroku buildpacks:add https://github.com/heroku/heroku-buildpack-python.git
Buildpack added. Next release on myproject will use:
  1. https://github.com/beanieboi/nginx-buildpack.git
  2. https://github.com/heroku/heroku-buildpack-pgbouncer.git
  3. https://github.com/heroku/heroku-buildpack-python.git
Run `git push heroku master` to create a new release using these buildpacks.
{% endhighlight %}

That's it for the buildpacks. Now let's have a look at how they work and configure the `Procfile` before we do that suggested push.

### Buildpack concepts
The Nginx buildpack's authors have already taken care of the configuration of Nginx so you won't have to do it.
You change things around using the Ruby instructions in the [repository](https://github.com/beanieboi/nginx-buildpack),
but I've found the default configuration to be working fine as is. You could set the `NGINX_WORKERS` environment variable, but it defaults to 4 which is about right for a regular Heroku dyno.

The pack uses three basic concepts:

* It starts with a shell script `bin/start-nginx`.
* It creates a unix socket `/tmp/nginx.socket` for your wsgi server.
* It doesn't accept requests until the file `/tmp/app-initialized` has been found.

Similarly the PgBouncer buildpack has many configuration options using `heroku config`.
The most useful is probably the `PGBOUNCER_MAX_CLIENT_CONN` setting, which I'm setting to our database servers plan maximum:
{% highlight bash %}
heroku config:set PGBOUNCER_MAX_CLIENT_CONN=120
{% endhighlight %}
You can check out all the other options at the [repository](https://github.com/heroku/heroku-buildpack-pgbouncer).
For it to work with your webserver and database, the buildpack uses these concepts:

* It starts with a shell script `bin/start-pgbouncer-stunnel`
* It will use the database connection set in your Heroku's `DATABASE_URL`
* On start it will replace `DATABASE_URL` with it's own connection information.

This means you don't have to change your Django database configuration as long as you were using the `DATABASE_URL` environment value, the [dj-database-url](https://github.com/kennethreitz/dj-database-url) or the compound [django-environ](http://django-environ.readthedocs.org/en/latest/) package.
I did run in some small problems with the url initialization though. Since it only does the old switcheroo once, any web workers that start before that will not have the new connection to PgBouncer. The same problem comes up when your web workers spawn from an environment other than the one that had its configuration updated.

### Procfile
Now it's time to put everything in a Procfile and get it running on Heroku. Since we use three different wsgi servers, I had to do this for each one seperately.
As you will see each server has it's unique way of doing this. All our setups contain a `web` entry which will be the wsgi server, behind Nginx, using PgBoucer for database connections.
The 'worker' entry will be our task queue worker for handling asynchronous and scheduled tasks. We use [Django Q](https://django-q.readthedocs.org), but it should also work with [Celery](http://www.celeryproject.org/) or any other worker of your choice that's going to be using lots of database connections.


#### Gunicorn
This is fairly straightforward. We use `start-nginx` to start `start-pgbouncer-stunnel`, which in turn should start gunicorn.
If anyone knows how to set the wsgi module in the configuration file, please leave a comment. I really don't like these long lines.

{% highlight bash %}
# Heroku with nginx, pgbouncer, gunicorn and django-q
web: bin/start-nginx bin/start-pgbouncer-stunnel gunicorn -c gunicorn.conf MyProject.wsgi:application
worker: bin/start-pgbouncer-stunnel python manage.py qcluster
{% endhighlight %}

Gunicorn has some very convenient server hooks , which we can override to touch the `tmp\app-initialized` file.
In this case I opted for the `when_ready` hook to tell Nginx to start accepting requests.

{% highlight python %}
# gunicorn.conf
def when_ready(server):
    # touch app-initialized when ready
    open('/tmp/app-initialized', 'w').close()

bind = 'unix:///tmp/nginx.socket'
workers = 4
{% endhighlight %}

That's really all there is to it. Now you can deploy and enjoy it.

### uWSGI
The same deal as with gunicorn. We chain the buildpack start commands and have uWSGI report to the Nginx socket.
Here I'm using the `hook-accepting1` hook, which is called when the first uWSGI worker is accepting connections.
Also two thumbs up for uWSGI for making it easy to execute `touch` or any other external command. The number of options uWSGI provides are mind boggling and this is both its appeal and its achilles heal. This is a basic configuration I know works well with Django and Heroku:  

{% highlight bash %}
# Heroku with nginx, pgbouncer, waitress and django-q
web: bin/start-nginx bin/start-pgbouncer-stunnel uwsgi uwsgi.ini
worker: bin/start-pgbouncer-stunnel python manage.py qcluster
{% endhighlight %}

{% highlight cfg %}
# uwsgi.ini
[uwsgi]
http-socket = /tmp/nginx.socket
master = true
processes = 4
die-on-term = true
memory-report = true
enable-threads = true
hook-accepting1 = exec:touch /tmp/app-initialized
env = DJANGO_SETTINGS_MODULE=MyProject.settings
module = MyProject.wsgi:application
{% endhighlight %}

### Waitress
We've been using [Waitress](http://docs.pylonsproject.org/projects/waitress/en/latest/) more and more for Django projects.
Orignally conceived for the Pylons project, this pure Python webserver uses multi-threading and is very well suited for the kind of long running queries you tend to get with large ORM databases. Even though it has its own request and response buffering, it can still benefit from Nginx's request handeling.

Waitress has no hooks, as far as I know. It is a Python module however so to get it to start Nginx, I wrote a little start script for it:

{% highlight bash %}
# Heroku with nginx, pgbouncer, waitress and django-q
web: bin/start-nginx bin/start-pgbouncer-stunnel python run.py
worker: bin/start-pgbouncer-stunnel python manage.py qcluster
{% endhighlight %}

{% highlight python %}
# run.py
from waitress import serve
from MyProject import wsgi

# touch app-initialized
open('/tmp/app-initialized', 'w').close()
# start waitress
serve(wsgi.application, unix_socket='/tmp/nginx.socket')
{% endhighlight %}