Django Nginx Memcache
=====================
Provides a view decorator. Use it to cache content in Memcache for nginx to
retrieve.

The cache keys are hashed using an md5 of the the request path *without*
GET parameters,

In progress 
-----------

IMPORTANT: If this has anything listed here, the code must be treated as unstable, even if it's on a master branch

#. Add support for generating cache keys based on full URIs, not just the path; this means the project can be used on shared servers, or servers serving multiple sites/subdomains which may have the same URI path


To do
-----

#. Add support for manually pushing a page via a function call
#. Add support for manually pushing a page via a signal
#. Add support for manually purging a page via a function call


Installation
------------

#. The usual pip or easy_install from `github <https://github.com/pcraciunoiu/django-nginx-memcache>`_::

    pip install -e git://github.com/pcraciunoiu/django-nginx-memcache#egg=django-nginx-memcache

#. Add ``nginx_memcache`` to your installed apps::

    INSTALLED_APPS = (
        # ...
        'nginx_memcache',
        # ...
    )

#. Then enable it and set the default cache timeout::

    CACHE_NGINX = True
    CACHE_NGINX_TIME = 3600 * 24  # 1 day, in seconds
    # Default backend to use from settings.CACHES
    # May need to update the nginx conf if this is changed
    CACHE_NGINX_ALIAS = 'default'

#. Setup Memcached appropriately as described in `Django's cache framework docs <http://docs.djangoproject.com/en/dev/topics/cache/#memcached>`_.

#. Install Nginx with the `set_misc <https://github.com/agentzh/set-misc-nginx-module>`_ or `set_hash module <https://github.com/simpl/ngx_http_set_hash>`_. This is required to compute md5 cache keys from within Nginx. (See installing nginx below).
#. Configure Nginx for direct Memcached page retrieval, i.e::

    # Nginx host configuration for demosite, using gunicorn
    #
    # Attempts to serve a page from memcache, falling
    # back to Django (via gunicorn) if it's not available 
                             
    upstream gunicorn_demosite {
             server 127.0.0.1:8003 fail_timeout=0;
    }

    server {
            listen 80 default_server;

            # Listen for all server names - lots of sites will be CNAMED
            # to this server, and we won't know what/which.
            # You could change this to a single server_name, of course

            server_name _;

            access_log /var/log/nginx/demosite.access.log;
            error_log /var/log/nginx/demosite.error.log;

            location /static/ {
                    root /usr/local/django/demosite/;
            }

            location /media/ {
                    # This example happens to be running in a virtualenv - update your path accordingly
                    root /usr/local/django/virtualenvs/demosite/lib/python2.7/site-packages/django/contrib/admin/;
            }

            location @gunicorn {
                    # This is the standard config for serving Django via gunicorn                                                                                                                            
                    root /usr/local/django/demosite/;

                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header Host $http_host;
                    proxy_redirect off;

                    if (!-f $request_filename) {
                        proxy_pass http://gunicorn_demosite;
                        break;
                    }

                    client_max_body_size 10m;
            }

            location @cache_miss {
                    # Pass on the request to gunicorn, creating
                    # a URI with the hostname as well as the path                                                                                                  
                    # See the docs if $is_args$args is confusing

                    set $caught_uri $http_host$uri$is_args$args;
                    try_files $caught_uri @gunicorn;
            }

            location / {
                    # By default, see if we can serve things from memcache.

                    # Extract cache key args and cache key.                                                                                                                                                 
                    if ($http_cookie ~* "pv=([^;]+)(?:;|$)") {
                            set $page_version $1;
                    }

                    # If you are running multiple sites off the same server, 
                    # the cache key to include the domain, too, which nginx
                    # doesn't consider part of the $uri. (SJ: it ought to do, but doesn't)

                    set_md5 $hash_key $http_host$uri&pv=$page_version;

                    set $memcached_key :1:$hash_key;

                    recursive_error_pages on;

                    set $fallthrough_uri null;

                    default_type       text/html;
                    memcached_pass     127.0.0.1:11211;
                    error_page         404 =200 @cache_miss; # cache misses get passed on as 200s (ideally)                                                          
                    error_page         401 $fallthrough_uri;
                    error_page         403 $fallthrough_uri;
                    error_page         405 $fallthrough_uri;

                    try_files $fallthrough_uri @cache_miss;
            }
    }

Installing Nginx
~~~~~~~~~~~~~~~~

These instructions apply for Ubuntu 11.04 and above::
`
    # install all dependencies
    sudo aptitude install libc6 libpcre3 libpcre3-dev libpcrecpp0 libssl0.9.8 libssl-dev zlib1g zlib1g-dev lsb-base

    # download nginx
    wget http://nginx.org/download/nginx-1.0.11.tar.gz
    tar -zxf nginx-1.0.11.tar.gz
    rm nginx-1.0.11.tar.gz
    cd nginx-1.0.11/

    # download modules
    wget https://github.com/simpl/ngx_devel_kit/zipball/v0.2.17 -O ngx_devel_kit.zip
    unzip ngx_devel_kit.zip
    wget https://github.com/agentzh/set-misc-nginx-module/zipball/v0.22rc4 -O set-misc-nginx-module.zip
    unzip set-misc-nginx-module.zip
    wget https://github.com/agentzh/echo-nginx-module/zipball/v0.37rc7 -O echo-nginx-module.zip
    unzip echo-nginx-module.zip

    # configure and install
    ./configure \
        --add-module=simpl-ngx_devel_kit-bc97eea \
        --add-module=agentzh-set-misc-nginx-module-290d6cb \
        --add-module=agentzh-echo-nginx-module-b7ea185 \
        --prefix=/usr \
        --pid-path=/var/run/nginx.pid \
        --lock-path=/var/lock/nginx.lock \
        --http-log-path=/var/log/nginx/access.log \
        --error-log-path=/var/log/nginx/error.log \
        --http-client-body-temp-path=/var/lib/nginx/body \
        --conf-path=/etc/nginx/nginx.conf \
        --with-http_flv_module \
        --with-http_ssl_module \
        --with-http_gzip_static_module \
        --http-proxy-temp-path=/var/lib/nginx/proxy \
        --with-http_stub_status_module \
        --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
        --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
        --http-scgi-temp-path=/var/lib/nginx/scgi
    make
    sudo make install

    # Done, now configure your nginx.


Usage
-----

nginx_memcache.decorators.cache_page_nginx
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``cache_page_nginx`` decorator caches the view's response content in Memcache. Any arguments are optional and outlined below.

Example::

    from nginx_memcache.decorators import cache_page_nginx

    @cache_page_nginx
    def my_view(request):
        ...

This will cache the view's response string in Memcache, and hereafter Nginx
will serve from Memcache directly, without hitting your Django server,
until the cache key expires.

Optional parameters
+++++++++++++++++++

``cache_timeout``
  Defaults to ``settings.CACHE_NGINX_TIME`` if not specified.

``page_version_fn``
  Use this to return a stringifiable version of the page, depending on the
  request. Example::

    def get_page_version(request):
        if request.user.is_authenticated():
            return 'authed'
        return 'anonymous'

``anonymous_only``
  Don't cache the page unless the user is anonymous, i.e. not authenticated.

Usage with forms and CSRF
~~~~~~~~~~~~~~~~~~~~~~~~~

If you want to embed forms on a cached page, you can leave out the context `{{ csrf() }}` or `{% csrf_token %}` and, instead, append it to all forms using JavaScript post page-load, or when a button is clicked.

Here's example JS and Django code for it::

    // JS code
    $.ajax({
        url: // your csrf url,
        type: 'GET',
        data: {type: 'login'},  // only if you need a session id for cookie login
        dataType: 'json',
        success: function(data) {
            $('form').each(function() {
                $(this).append(
                    '<input type=hidden name=csrfmiddlewaretoken ' +
                        ' value="' + data.token + '">');
            });
        }
    });

    // Django code
    # views.py, don't forget to add to urls.py
    def get_csrf(request):
        if request.GET.get('type') == 'login':
            request.session.set_test_cookie()
        return JSONResponse({
            'status': 1,
            'token': getattr(request, 'csrf_token', 'NOTPROVIDED')
        })


Full List of Settings
~~~~~~~~~~~~~~~~~~~~~

``CACHE_NGINX``
  Set this to False to disable any caching. E.g. for testing, staging...

``CACHE_NGINX_TIME``
  Default cache timeout.

``CACHE_NGINX_ALIAS``
  Which cache backend to use from `settings.CACHES <https://docs.djangoproject.com/en/dev/ref/settings/#std:setting-CACHES>`_

Contributing
============
If you'd like to fix a bug, add a feature, etc

#. Start by opening an issue.
    Be explicit so that project collaborators can understand and reproduce the
    issue, or decide whether the feature falls within the project's goals.
    Code examples can be useful, too.

#. File a pull request.
    You may write a prototype or suggested fix.

#. Check your code for errors, complaints.
    Use `check.py <https://github.com/jbalogh/check>`_

#. Write and run tests.
    Write your own test showing the issue has been resolved, or the feature
    works as intended.

Running Tests
=============
To run the tests::

    python manage.py test nginx_memcache