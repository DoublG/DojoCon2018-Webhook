=======
Webhook
=======
The webhook application was build for the *EnviromentalSensor* project, presented on DojoCon2018 (Belgium).
I acts as an entrypoint to internal RabbitMQ servers, it is used to communicate between
Electron devices via the Particle Cloud and an internal Node-Red server with a dashboard.

.. image:: overview.jpg
   :alt: Global overview
   :align: center

-------------
Configuration
-------------
Configuration is loaded via a configuration file supplied
via the *APP_SETTINGS* environment variable.

Mandatory values in the config file.

=========================== =========================================
Configuration name          Description
=========================== =========================================
*GOOGLE_KEY*                Google API key
*API_KEY*                   "API Password" for webhook authorisation
*RABBITMQ_HOST*             RabbitMQ host ip
*RABBITMQ_USER*             RabbitMQ username
*RABBITMQ_PWD*              RabbitMQ password
*RABBITMQ_EXCHANGE*         RabbitMQ exchange name
*RABBITMQ_EXCHANGE_TYPE*    RabbitMQ exchange type
=========================== =========================================

URL's
=====
All url's use the POST method, the methods only return a success message,
the "real" response is published under *RABBITMQ_EXCHANGE*
with routing key *<routing_key>*.

=========================== ==========================================
Configuration name          Description
=========================== ==========================================
*/<routing_key>/street*     Get the nearest street
*/<routing_key>/geo*        Get the location based on GSM network info
*/<routing_key>*            Publish JSON to RabbitMQ exchange
=========================== ==========================================

Server config files
===================
Backend Server configuration
----------------------------

The configuration expects the following folders to be created,
these have to be created afterwards.

+----------------------------------+
|paths                             |
+==================================+
|/etc/nginx/applications-available |
+----------------------------------+
|/etc/nginx/applications-enabled   |
+----------------------------------+
|/etc/uwsgi/vassals                |
+----------------------------------+

NGINX
`````
**/etc/nginx/nginx.conf** ::

    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;

    events {
            worker_connections 768;
    }

    http {
            sendfile on;
            tcp_nopush on;
            tcp_nodelay on;
            keepalive_timeout 65;
            types_hash_max_size 2048;

            include /etc/nginx/mime.types;
            default_type application/octet-stream;

            ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
            ssl_prefer_server_ciphers on;

            access_log /var/log/nginx/access.log;
            error_log /var/log/nginx/error.log;

            gzip on;
            gzip_disable "msie6";

            include /etc/nginx/conf.d/*.conf;
            include /etc/nginx/sites-enabled/*;
    }

**/etc/nginx/sites-enabled/applications** ::

    server {
      listen 5051 ssl default_server;

      server_name rabbitmq;

      ssl_certificate     /root/CA/keys/rabbitmq.crt;
      ssl_certificate_key /root/CA/keys/rabbitmq.key;
      ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers         HIGH:!aNULL:!MD5;

      include /etc/nginx/applications-enabled/*;
    }

**/etc/nginx/applications-available/template** ::

    location /%n/ {
      include /etc/nginx/uwsgi_params;
      rewrite ^/%n/(.*)$ /$1 break;
      uwsgi_pass unix:/var/www/%n/%n.socket;
    }


UWSGI
`````
**/etc/default/uwsgi** ::

    RUN_AT_STARTUP=yes
    VERBOSE=yes
    PRINT_CONFNAMES_IN_INITD_SCRIPT_OUTPUT=no
    INHERITED_CONFIG=/etc/uwsgi/config.ini


**/etc/uwsgi/config.ini** ::

    [uwsgi]
    autoload = true
    master = true
    workers = 2
    no-orphans = true
    pidfile = /run/uwsgi/%(deb-confnamespace)/%(deb-confname)/pid
    socket = /run/uwsgi/%(deb-confnamespace)/%(deb-confname)/socket
    chmod-socket = 660
    log-date = true

**/etc/uwsgi/apps-available/emperor.ini** ::

    [uwsgi]
    emperor = /etc/uwsgi/vassals/*.ini
    emperor-use-clone = fs,ipc,pid,uts

**/etc/uwsgi/apps-available/template.ini** ::

    [uwsgi]
    socket = /var/www/%n/%n.socket
    module = %n:create_app()
    chdir = /var/www/%n
    home = /var/www/%n
    env = APP_SETTINGS=/var/www/%n/config.cfg
    virtualenv = /var/www/%n/env
    plugins=python3
    vacuum = true
    uid=www-%n
    guid=www-%n

RabbitMQ
````````
TODO

Frontend Server configuration
-----------------------------
The configuration expects the following folders to be created,
these have to be created afterwards.

+----------------------------------+
|paths                             |
+==================================+
|/etc/nginx/stream-available       |
+----------------------------------+
|/etc/nginx/stream-enabled         |
+----------------------------------+
NGINX
`````
**/etc/nginx/stream-enabled/rabbitmq-amqp** ::

    server {
      listen 6666 ssl;

      proxy_ssl on;
      proxy_ssl_trusted_certificate /etc/ssl/certs/remote_ca.crt;
      proxy_ssl_verify on;
      proxy_ssl_session_reuse on;


      ssl_certificate     /etc/letsencrypt/live/****.be/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/****.be/privkey.pem;
      ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers         HIGH:!aNULL:!MD5;

      proxy_pass rabbitmq:5671;
    }


**/etc/nginx/stream-enabled/rabbitmq-mqtt** ::

    server {
      listen 7777 ssl;

      proxy_ssl_verify on;
      proxy_ssl on;
      proxy_ssl_trusted_certificate /etc/ssl/certs/remote_ca.crt;
      proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      proxy_ssl_ciphers HIGH:!aNULL:!MD5;

      ssl_certificate     /etc/letsencrypt/live/****.be/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/****.be/privkey.pem;
      ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers         HIGH:!aNULL:!MD5;

      proxy_pass rabbitmq:8883;
    }

**/etc/nginx/sites-enabled/main** ::

    server {
        listen 5051 ssl;
        listen [::]:5051 ssl;

        server_name ****.be;

        ssl_certificate     /etc/letsencrypt/live/****.be/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/****.be/privkey.pem;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        root /var/www/html;

        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_ssl_trusted_certificate /etc/ssl/certs/remote_ca.crt;
            proxy_pass https://rabbitmq:5051;
            proxy_ssl_verify on;
            proxy_intercept_errors on;
            error_page 404 401 =444 /;
        }
    }

    server {
        listen 80;
        server_name ****.be;
        root /var/www/html;
    }

**/etc/nginx/nginx.conf** ::

    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;

    events {
            worker_connections 768;
    }

    http {

            sendfile on;
            tcp_nopush on;
            tcp_nodelay on;
            keepalive_timeout 65;
            types_hash_max_size 2048;

            include /etc/nginx/mime.types;
            default_type application/octet-stream;

            ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
            ssl_prefer_server_ciphers on;

            access_log /var/log/nginx/access.log;
            error_log /var/log/nginx/error.log;

            gzip on;
            gzip_disable "msie6";

            include /etc/nginx/conf.d/*.conf;
            include /etc/nginx/sites-enabled/*;
    }


    stream{
      include /etc/nginx/stream-enabled/*;
    }


-------------
Authorisation
-------------
All url's are protected by a simple API key, for every call you need to
supply this key, you can pick one of the supported methods.

============= ==================
Name          Location
============= ==================
*api_key*     GET HTTP attribute
*X-API-Key*   HTTP Header
*X-API-KEY*   Cookie name
============= ==================

-------------------------------
SSL Termination & Re-encryption
-------------------------------
EasyEncypt
============
* Create CA
* Create server key

----------------
Request examples
----------------

POST: /<routing_key>/geo
========================
**Example geo request:** ::

    {
      "homeMobileCountryCode": 206,
      "homeMobileNetworkCode": 1,
      "considerIp": false,
      "carrier": "Proximus",
      "cellTowers": [
        {
          "cellId": 66674698,
          "locationAreaCode": 3024,
          "mobileCountryCode": 206,
          "mobileNetworkCode": 1
        },
        {
          "cellId": 46190596,
          "locationAreaCode": 3052,
          "mobileCountryCode": 206,
          "mobileNetworkCode": 1
        },
        {
          "cellId": 21409538,
          "locationAreaCode": 3052,
          "mobileCountryCode": 206,
          "mobileNetworkCode": 1
        }
      ]
    }

POST: /<routing_key>/street
============================
**Example street request:** ::

    {'long': 4.8367074, 'lat': 51.321642499999996 }

POST: /<routing_key>
====================
**Example normal request:** ::

  Any valid json is allowed

----------
Deployment
----------
I'm using fabric for the deployment, in the examples 100.100.0.2 (rabbitmq) is the backend server and 100.100.0.1 is the frontend server.

**cleanup of the previous setup** ::

    fab -H root@100.100.0.2 cleanup-application

**update / install new application** ::

    fab -H root@100.100.0.2 build-application

**check installation** ::

    fab -H root@100.100.0.2 check-installation

