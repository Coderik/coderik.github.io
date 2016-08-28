---
layout: post
title:  "Setting up Nginx + uWSGI + Flask"
date:   2016-08-23 14:55:00 +0200
categories: nginx flask
---
This is just a brief note on how to setup Nginx + uWSGI + Flask on a freshly installed Ubuntu server.
We are going to start from uWSGI, then add Flask, and finally install and configure Nginx web server. 

<!--more-->

### Step one: uWSGI

First of all let's install everything that is needed:

{% highlight Bash %}
sudo apt-get install python-dev
sudo apt-get install python-pip
sudo pip install virtualenv
{% endhighlight %}

Setup application folder:

{% highlight Bash %}
cd <app-path>
mkdir <app-name>
mkdir <app-name>/app
mkdir <app-name>/app/static
mkdir <app-name>/app/templates
cd <app-name>
{% endhighlight %}

Init and activate virtual environment:

{% highlight Bash %}
virtualenv env
source env/bin/activate
{% endhighlight %}

install uWSGI:

{% highlight Bash %}
pip install uwsgi
{% endhighlight %}

To test uWSGI, create uwsgi_test.py file:

{% highlight Python %}
# uwsgi_test.py
def application(env, start_response):
     start_response('200 OK', [('Content-Type','text/html')])
     return ["Hello from uWSGI"] # python2
     #return [b"Hello from uWSGI"] # python3
{% endhighlight %}

...and run:

{% highlight Bash %}
uwsgi --http :8080 --wsgi-file uwsgi_test.py
{% endhighlight %}

Go to localhost:8080.
If works, then we have:

**web-client <-> uWSGI <-> Python**

To cleanup we can remove uwsgi_test.py.


### Step two: Flask

Install Flask:

{% highlight Bash %}
pip install flask
{% endhighlight %}

To test Flask, create file app/app.py:

{% highlight Python %}
# app.py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
     return "Flask works!"

if __name__ == "__main__":
     app.run()
{% endhighlight %}

...and run:

{% highlight Bash %}
uwsgi --http :8080 --wsgi-file app/app.py --callable app
{% endhighlight %}

Go to localhost:8080.
If works, then we have: 

**web-client <-> uWSGI <-> Flask**


### Step three: Nginx

Install the necessary software and dependencies:

{% highlight Bash %}
# for compiler and git
sudo apt-get install git gcc make
# for the HTTP rewrite module which requires the PCRE library
sudo apt-get install libpcre3-dev
# for SSL modules
sudo apt-get install libssl-dev
{% endhighlight %}

Download the Nginx source files to <nginx-src>:
{% highlight Bash %}
cd <nginx-src>
mkdir nginx
cd nginx
wget http://nginx.org/download/nginx-1.9.5.tar.gz
tar zxpvf nginx-1.9.5.tar.gz
cd nginx-1.9.5
{% endhighlight %}

If you are not going to recompile Nginx again and again, just set <nginx-src> to ~/Downloads.

Compile Nginx:
{% highlight Bash %}
./configure --with-http_ssl_module --prefix=/usr/local/nginx/
sudo make
sudo make install
{% endhighlight %}

Notice that the *prefix* is set to /usr/local/nginx/. This is ok for developer's machine; however, on a server you would probably prefer to put Nginx in another location.

Now it is time to configure Nginx:

{% highlight Bash %}
cd /usr/local/nginx/conf
sudo cp nginx.conf nginx.cong.old
sudo sublime nginx.conf
{% endhighlight %}

Modify nginx.cong (only the essential part is shown):

{% highlight Bash %}
http {
    upstream flask {
        server unix:/tmp/<app-name>.sock;  # for file socket
    }

    server {
        listen 80;
        server_name localhost;

        # Proxying connections to the application server
        location / {
            include uwsgi_params;
            uwsgi_pass flask;
        }

        # Setting to by-pass for static files
        location /static {
            alias /<app-path>/<app-name>/app/static;
        }
    }
}
{% endhighlight %}

Reload the config file:

{% highlight Bash %}
cd /usr/local/nginx/sbin/
sudo ./nginx -s reload
{% endhighlight %}


### Step three: running uWSGI

To test Nginx - uWSGI binding let's first run uwsgi manually:

{% highlight Bash %}
cd <app-path>/<app-name>
uwsgi --socket /tmp/<app-name>.sock --wsgi-file app/app.py --callable app
{% endhighlight %}

If this does not work, probably it is socket's permission issue. Try:
{% highlight Bash %}
uwsgi --socket /tmp/<app-name>.sock --wsgi-file app/app.py --callable app --chmod-socket=660
{% endhighlight %}

Go to localhost.
If works, then we have: 

**web-client <-> Nginx <-> socket <-> uWSGI <-> Flask**

Now we can configure uWSGI to be launched by Upstart system.

Create file <app-name>.ini

{% highlight Bash %}
[uwsgi]
module = app:app

master = true
processes = 5

socket = /tmp/<app-name>.sock
chmod-socket = 666
vacuum = true

die-on-term = true
{% endhighlight %}

Create an Upstart script. Creating an Upstart script will allow Ubuntu's init system to automatically start uWSGI and serve our Flask application whenever the server boots.

Create a script file <app-name>.conf within the /etc/init directory to begin:

{% highlight Bash %}
# /etc/init/<app-name>.conf
description "uWSGI server instance serving <app-name>"

start on runlevel [2345]
stop on runlevel [!2345]

setuid <user>
setgid <group>

env PATH=<app-path>/<app-name>/env/bin
chdir <app-path>/<app-name>/app
exec uwsgi --ini <app-name>.ini
{% endhighlight %}

Now we can start the process:

{% highlight Bash %}
sudo start <app-name>
{% endhighlight %}

In order to restart the process simply run:

{% highlight Bash %}
sudo restart <app-name>
{% endhighlight %}


### References
Exactly the same procedure is described (and probably better) in many other places:
[one](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-14-04),
[two](https://www.digitalocean.com/community/tutorials/how-to-deploy-python-wsgi-applications-using-uwsgi-web-server-with-nginx),
[three](http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html).

Running Nginx with Upstart is described [here](https://www.nginx.com/resources/wiki/start/topics/examples/ubuntuupstart/#).

