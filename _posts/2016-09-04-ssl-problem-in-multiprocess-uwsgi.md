---
layout: post
title:  "SSL problem in multiprocess uWSGI app"
date:   2016-09-04 17:31:00 +0200
categories: uwsgi
---
I have encountered a tricky issue while playing with [python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot) library. A clear sign of the issue is the following exception appearing from time to time when we send a message to Telegram via python-telegram-bot:

{% highlight Bash %}
urllib3 HTTPError [Errno 1] _ssl.c:1429: error:1408F119:SSL routines:SSL3_GET_RECORD:decryption failed or bad record mac
{% endhighlight %}

Well, actually the issue has nothing to do with python-telegram-bot library itself, but is rather related to the implementation of OpenSSL in Debian. The problem appears when *the same* SSL connection is used in two concurrent processes. As always with concurrency-related issues, everything might work correctly for a while, then break suddenly and mysteriously, and then continue to work for another hundreds of trials. Leaving the deep internals of SSl aside, let's see how it affects a telegram bot.

<!--more-->

Commonly uWSGI is configured to spawn multiple working processes to serve requests. Every process runs its own instance of our Flask application. What causes the problem is the way uWSGI spawns these processes by default. It initializes an application only once and then copies it to several working processes. This saves some memory, but unfortunately is not appropriate in our case. Most likely in our bot application we create telegram.Bot object only once - on initialization - and end up with the same instance in all the processes. Shared telegram.Bot object means shared SSL connection.

In order to change the default behavior we should set *lazy-apps* option in uWSGI config. Note that there is another option called *lazy*, but its usage is strongly discouraged in the [doc](http://uwsgi-docs.readthedocs.io/en/latest/ThingsToKnow.html).

To sum it up, the uWSGI config *your-app-name.ini* might look somewhat like this:

{% highlight Bash %}
[uwsgi]
module = <FILE:CALLABLE>

master = true
processes = 2

socket = <SOCKET>
chmod-socket = 666
vacuum = true

# Fix 'ssl+multiprocessing' issue
lazy-apps = true

die-on-term = true
{% endhighlight %}

### Relevant references:
* [Issue in "requests" lib's tracker](https://github.com/kennethreitz/requests/issues/1906)
* [Stackoverflow](http://stackoverflow.com/questions/3724900/python-ssl-problem-with-multiprocessing)
* [Stackoverflow](http://stackoverflow.com/questions/22752521/uwsgi-flask-sqlalchemy-and-postgres-ssl-error-decryption-failed-or-bad-reco)

