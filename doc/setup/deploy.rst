:banner: banners/flectra_deploying.jpg

=================
Deploying Flectra
=================

This document describes basic steps to set up Flectra in production or on an
internet-facing server. It follows :ref:`installation <setup/install>`, and is
not generally necessary for a development systems that is not exposed on the
internet.

.. warning:: If you are setting up a public server, be sure to check our :ref:`security` recommandations!


.. _db_filter:

dbfilter
========

Flectra is a multi-tenant system: a single Flectra system may run and serve a number
of database instances. It is also highly customizable, with customizations
(starting from the modules being loaded) depending on the "current database".

This is not an issue when working with the backend (web client) as a logged-in
company user: the database can be selected when logging in, and customizations
loaded afterwards.

However it is an issue for non-logged users (portal, website) which aren't
bound to a database: Flectra needs to know which database should be used to load
the website page or perform the operation. If multi-tenancy is not used that is not an
issue, there's only one database to use, but if there are multiple databases
accessible Flectra needs a rule to know which one it should use.

That is one of the purposes of :option:`--db-filter <flectra-bin --db-filter>`:
it specifies how the database should be selected based on the hostname (domain)
that is being requested. The value is a `regular expression`_, possibly
including the dynamically injected hostname (``%h``) or the first subdomain
(``%d``) through which the system is being accessed.

For servers hosting multiple databases in production, especially if ``website``
is used, dbfilter **must** be set, otherwise a number of features will not work
correctly.

Configuration samples
---------------------

* Show only databases with names beginning with 'mycompany'

in ``/etc/flectra.conf`` set:

.. code-block:: ini

  [options]
  dbfilter = ^mycompany.*$

* Show only databases matching the first subdomain after ``www``: for example
  the database "mycompany" will be shown if the incoming request
  was sent to ``www.mycompany.com`` or ``mycompany.co.uk``, but not
  for ``www2.mycompany.com`` or ``helpdesk.mycompany.com``.

in ``/etc/flectra.conf`` set:

.. code-block:: ini

  [options]
  dbfilter = ^%d$

.. note::
  Setting a proper :option:`--db-filter <flectra-bin --db-filter>` is an important part
  of securing your deployment.
  Once it is correctly working and only matching a single database per hostname, it
  is strongly recommended to block access to the database manager screens,
  and to use the ``--no-database-list`` startup paramater to prevent listing
  your databases, and to block access to the database management screens.
  See also security_.


PostgreSQL
==========

By default, PostgreSQL only allows connection over UNIX sockets and loopback
connections (from "localhost", the same machine the PostgreSQL server is
installed on).

UNIX socket is fine if you want Flectra and PostgreSQL to execute on the same
machine, and is the default when no host is provided, but if you want Flectra and
PostgreSQL to execute on different machines [#different-machines]_ it will
need to `listen to network interfaces`_ [#remote-socket]_, either:

* Only accept loopback connections and `use an SSH tunnel`_ between the
  machine on which Flectra runs and the one on which PostgreSQL runs, then
  configure Flectra to connect to its end of the tunnel
* Accept connections to the machine on which Flectra is installed, possibly
  over ssl (see `PostgreSQL connection settings`_ for details), then configure
  Flectra to connect over the network

Configuration sample
--------------------

* Allow tcp connection on localhost
* Allow tcp connection from 192.168.1.x network

in ``/etc/postgresql/9.5/main/pg_hba.conf`` set:

.. code-block:: text

  # IPv4 local connections:
  host    all             all             127.0.0.1/32            md5
  host    all             all             192.168.1.0/24          md5

in ``/etc/postgresql/9.5/main/postgresql.conf`` set:
  
.. code-block:: text
  
  listen_addresses = 'localhost,192.168.1.2'
  port = 5432
  max_connections = 80

.. _setup/deploy/flectra:

Configuring Flectra
-------------------

Out of the box, Flectra connects to a local postgres over UNIX socket via port
5432. This can be overridden using :ref:`the database options
<reference/cmdline/server/database>` when your Postgres deployment is not
local and/or does not use the installation defaults.

The :ref:`packaged installers <setup/install/packaged>` will automatically
create a new user (``flectra``) and set it as the database user.

* The database management screens are protected by the ``admin_passwd``
  setting. This setting can only be set using configuration files, and is
  simply checked before performing database alterations. It should be set to
  a randomly generated value to ensure third parties can not use this
  interface.
* All database operations use the :ref:`database options
  <reference/cmdline/server/database>`, including the database management
  screen. For the database management screen to work requires that the PostgreSQL user
  have ``createdb`` right.
* Users can always drop databases they own. For the database management screen
  to be completely non-functional, the PostgreSQL user needs to be created with
  ``no-createdb`` and the database must be owned by a different PostgreSQL user.

  .. warning:: the PostgreSQL user *must not* be a superuser

Configuration sample
~~~~~~~~~~~~~~~~~~~~

* connect to a PostgreSQL server on 192.168.1.2
* port 5432
* using an 'flectra' user account,
* with 'pwd' as a password
* filtering only db with a name beginning with 'mycompany'

in ``/etc/flectra.conf`` set:

.. code-block:: ini

  [options]
  admin_passwd = mysupersecretpassword
  db_host = 192.168.1.2
  db_port = 5432
  db_user = flectra
  db_password = pwd
  dbfilter = ^mycompany.*$

.. _builtin_server:

Builtin server
==============

Flectra includes built-in HTTP servers, using either multithreading or
multiprocessing.

For production use, it is recommended to use the multiprocessing server as it
increases stability, makes somewhat better use of computing resources and can
be better monitored and resource-restricted.

* Multiprocessing is enabled by configuring :option:`a non-zero number of
  worker processes <flectra-bin --workers>`, the number of workers should be based
  on the number of cores in the machine (possibly with some room for cron
  workers depending on how much cron work is predicted)
* Worker limits can be configured based on the hardware configuration to avoid
  resources exhaustion

.. warning:: multiprocessing mode currently isn't available on Windows


Worker number calculation
-------------------------

* Rule of thumb : (#CPU * 2) + 1
* Cron workers need CPU
* 1 worker ~= 6 concurrent users

memory size calculation
-----------------------

* We consider 20% of the requests are heavy requests, while 80% are simpler ones
* A heavy worker, when all computed field are well designed, SQL requests are well designed, ... is estimated to consume around 1Go of RAM
* A lighter worker, in the same scenario, is estimated to consume around 150MB of RAM

Needed RAM = #worker * ( (light_worker_ratio * light_worker_ram_estimation) + (heavy_worker_ratio * heavy_worker_ram_estimation) )

LiveChat
--------

In multiprocessing, a dedicated LiveChat worker is automatically started and
listening on :option:`the longpolling port <flectra-bin --longpolling-port>` but
the client will not connect to it.

Instead you must have a proxy redirecting requests whose URL starts with
``/longpolling/`` to the longpolling port. Other request should be proxied to
the :option:`normal HTTP port <flectra-bin --http-port>`

Configuration sample
--------------------

* Server with 4 CPU, 8 Thread
* 60 concurrent users

* 60 users / 6 = 10 <- theorical number of worker needed
* (4 * 2) + 1 = 9 <- theorical maximal number of worker
* We'll use 8 workers + 1 for cron. We'll also use a monitoring system to measure cpu load, and check if it's between 7 and 7.5 .
* RAM = 9 * ((0.8*150) + (0.2*1024)) ~= 3Go RAM for Flectra

in ``/etc/flectra.conf``:

.. code-block:: ini

  [options]
  limit_memory_hard = 1677721600
  limit_memory_soft = 629145600
  limit_request = 8192
  limit_time_cpu = 600
  limit_time_real = 1200
  max_cron_threads = 1
  workers = 8

.. _https_proxy:

HTTPS
=====

Whether it's accessed via website/web client or web service, Flectra transmits
authentication information in cleartext. This means a secure deployment of
Flectra must use HTTPS\ [#switching]_. SSL termination can be implemented via
just about any SSL termination proxy, but requires the following setup:

* Enable Flectra's :option:`proxy mode <flectra-bin --proxy-mode>`. This should only be enabled when Flectra is behind a reverse proxy
* Set up the SSL termination proxy (`Nginx termination example`_)
* Set up the proxying itself (`Nginx proxying example`_)
* Your SSL termination proxy should also automatically redirect non-secure
  connections to the secure port

.. warning::

  In case you are using the Point of Sale module in combination with a `POSBox`_,
  you must disable the HTTPS configuration for the route ``/pos/web`` to avoid
  mixed-content errors.

Configuration sample
--------------------

* Redirect http requests to https
* Proxy requests to flectra

in ``/etc/flectra.conf`` set:

.. code-block:: ini

  proxy_mode = True

in ``/etc/nginx/sites-enabled/flectra.conf`` set:

.. code-block:: nginx

  #flectra server
  upstream flectra {
   server 127.0.0.1:7073;
  }
  upstream flectrachat {
   server 127.0.0.1:8072;
  }
  
  # http -> https
  server {
     listen 80;
     server_name flectra.mycompany.com;
     rewrite ^(.*) https://$host$1 permanent;
  }
  
  server {
   listen 443 ssl http2;
   server_name flectra.mycompany.com;
   proxy_read_timeout 720s;
   proxy_connect_timeout 720s;
   proxy_send_timeout 720s;


   # Add Headers for flectra proxy mode
  proxy_set_header Host $host;
  proxy_set_header remote_addr $remote_addr;
  proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
  
   proxy_set_header X-Forwarded-Host $host;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto $scheme;
   proxy_set_header X-Real-IP $remote_addr;

  proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
  proxy_redirect off;
  proxy_buffering off;
   
   # SSL parameters
   ssl on;
   ssl_certificate /etc/ssl/nginx/server.crt;
   ssl_certificate_key /etc/ssl/nginx/server.key;
   ssl_session_timeout 30m;
   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
   ssl_prefer_server_ciphers on;
   
   # log
   access_log /var/log/nginx/flectra.access.log;
   error_log /var/log/nginx/flectra.error.log;
   
   # Redirect requests to flectra backend server
   location / {
     proxy_redirect off;
     proxy_pass http://flectra;
   }
   location /longpolling {
       proxy_pass http://flectrachat;
   }
 
   # common gzip
   gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
   gzip on;
  }
 
Flectra as a WSGI Application
=============================

It is also possible to mount Flectra as a standard WSGI_ application. Flectra
provides the base for a WSGI launcher script as ``flectra-wsgi.example.py``. That
script should be customized (possibly after copying it from the setup directory) to correctly set the
configuration directly in :mod:`flectra.tools.config` rather than through the
command-line or a configuration file.

However the WSGI server will only expose the main HTTP endpoint for the web
client, website and webservice API. Because Flectra does not control the creation
of workers anymore it can not setup cron or livechat workers

Cron Workers
------------

To run cron jobs for an Flectra deployment as a WSGI application requires

* A classical Flectra (run via ``flectra-bin``)
* Connected to the database in which cron jobs have to be run (via
  :option:`flectra-bin -d`)
* Which should not be exposed to the network. To ensure cron runners are not
  network-accessible, it is possible to disable the built-in HTTP server
  entirely with :option:`flectra-bin --no-http` or setting ``http_enable = False``
  in the configuration file

LiveChat
--------

The second problematic subsystem for WSGI deployments is the LiveChat: where
most HTTP connections are relatively short and quickly free up their worker
process for the next request, LiveChat require a long-lived connection for
each client in order to implement near-real-time notifications.

This is in conflict with the process-based worker model, as it will tie
up worker processes and prevent new users from accessing the system. However,
those long-lived connections do very little and mostly stay parked waiting for
notifications.

The solutions to support livechat/motifications in a WSGI application are:

* Deploy a threaded version of Flectra (instread of a process-based preforking
  one) and redirect only requests to URLs starting with ``/longpolling/`` to
  that Flectra, this is the simplest and the longpolling URL can double up as
  the cron instance.
* Deploy an evented Flectra via ``flectra-gevent`` and proxy requests starting
  with ``/longpolling/`` to
  :option:`the longpolling port <flectra-bin --longpolling-port>`.

Serving Static Files
====================

For development convenience, Flectra directly serves all static files in its
modules. This may not be ideal when it comes to performances, and static
files should generally be served by a static HTTP server.

Flectra static files live in each module's ``static/`` folder, so static files
can be served by intercepting all requests to :samp:`/{MODULE}/static/{FILE}`,
and looking up the right module (and file) in the various addons paths.

.. todo:: test whether it would be interesting to serve filestored attachments
          via this, and how (e.g. possibility of mapping ir.attachment id to
          filestore hash in the database?)

.. _security:

Security
========

For starters, keep in mind that securing an information system is a continuous process,
not a one-shot operation. At any moment, you will only be as secure as the weakest link
in your environment.

So please do not take this section as the ultimate list of measures that will prevent
all security problems. It's only intended as a summary of the first important things
you should be sure to include in your security action plan. The rest will come
from best security practices for your operating system and distribution,
best practices in terms of users, passwords, and access control management, etc.

When deploying an internet-facing server, please be sure to consider the following
security-related topics:

- Always set a strong super-admin admin password, and restrict access to the database
  management pages as soon as the system is set up. See :ref:`db_manager_security`.

- Choose unique logins and strong passwords for all administrator accounts on all databases.
  Do not use 'admin' as the login. Do not use those logins for day-to-day operations,
  only for controlling/managing the installation.
  *Never* use any default passwords like admin/admin, even for test/staging databases.

- Use appropriate database filters ( :option:`--db-filter <flectra-bin --db-filter>`)
  to restrict the visibility of your databases according to the hostname.
  See :ref:`db_filter`.
  You may also use :option:`-d <flectra-bin -d>` to provide your own (comma-separated)
  list of available databases to filter from, instead of letting the system fetch
  them all from the database backend.

- Once your ``db_name`` and ``db_filter`` are configured and only match a single database
  per hostname, you should set ``list_db`` configuration option to ``False``, to prevent
  listing databases entirely, and to block access to the database management screens
  (this is also exposed as the :option:`--no-database-list <flectra-bin --no-database-list>`
  command-line option)

- Make sure the PostgreSQL user (:option:`--db_user <flectra-bin --db_user>`) is *not* a super-user,
  and that your databases are owned by a different user. For example they could be owned by
  the ``postgres`` super-user if you are using a dedicated non-privileged ``db_user``.
  See also :ref:`setup/deploy/flectra`.

- Keep installations updated by regularly installing the latest builds,
  either via GitHub or by downloading the latest version from
  https://flectrahq.com/download

- Configure your server in multi-process mode with proper limits matching your typical
  usage (memory/CPU/timeouts). See also :ref:`builtin_server`.

- Run Flectra behind a web server providing HTTPS termination with a valid SSL certificate,
  in order to prevent eavesdropping on cleartext communications. SSL certificates are
  cheap, and many free options exist.
  Configure the web proxy to limit the size of requests, set appropriate timeouts,
  and then enable the :option:`proxy mode <flectra-bin --proxy-mode>` option.
  See also :ref:`https_proxy`.

- Whenever possible, host your public-facing demo/test/staging instances on different
  machines than the production ones. And apply the same security precautions as for
  production.

- If you are hosting multiple customers, isolate customer data and files from each other
  using containers or appropriate "jail" techniques.

- Setup daily backups of your databases and filestore data, and copy them to a remote
  archiving server that is not accessible from the server itself.


.. _db_manager_security:

Database Manager Security
-------------------------

:ref:`setup/deploy/flectra` mentioned ``admin_passwd`` in passing.

This setting is used on all database management screens (to create, delete,
dump or restore databases).

If the management screens must not be accessible at all, you should set ``list_db``
configuration option to ``False``, to block access to all the database selection and
management screens. But be sure to setup an appropriate ``db_name`` parameter
(and optionally, ``db_filter`` too) so that the system can determine the target database
for each request, otherwise users will be blocked as they won't be allowed to choose the
database themselves.

If the management screens must only be accessible from a selected set of machines,
use the proxy server's features to block access to all routes starting with ``/web/database``
except (maybe) ``/web/database/selector`` which displays the database-selection screen.

If the database-management screen should be left accessible, the
``admin_passwd`` setting must be changed from its ``admin`` default: this
password is checked before allowing database-alteration operations.

It should be stored securely, and should be generated randomly e.g.

.. code-block:: console

    $ python3 -c 'import base64, os; print(base64.b64encode(os.urandom(24)))'

which will generate a 32 characters pseudorandom printable string.

Supported Browsers
==================

Flectra is supported by multiple browsers for each of its versions. No 
distinction is made according to the browser version in order to be
up-to-date. Flectra is supported on the current browser version. The list 
of the supported browsers by Flectra version is the following:

- **Flectra 1.0+:** Mozilla Firefox, Google Chrome, Safari, Microsoft Edge

.. [#different-machines]
    to have multiple Flectra installations use the same PostgreSQL database,
    or to provide more computing resources to both software.
.. [#remote-socket]
    technically a tool like socat_ can be used to proxy UNIX sockets across
    networks, but that is mostly for software which can only be used over
    UNIX sockets
.. [#switching]
    or be accessible only over an internal packet-switched network, but that
    requires secured switches, protections against `ARP spoofing`_ and
    precludes usage of WiFi. Even over secure packet-switched networks,
    deployment over HTTPS is recommended, and possible costs are lowered as
    "self-signed" certificates are easier to deploy on a controlled
    environment than over the internet.

.. _regular expression: https://docs.python.org/3/library/re.html
.. _ARP spoofing: http://en.wikipedia.org/wiki/ARP_spoofing
.. _Nginx termination example:
    http://nginx.com/resources/admin-guide/nginx-ssl-termination/
.. _Nginx proxying example:
    http://nginx.com/resources/admin-guide/reverse-proxy/
.. _socat: http://www.dest-unreach.org/socat/
.. _PostgreSQL connection settings:
.. _listen to network interfaces:
    http://www.postgresql.org/docs/9.3/static/runtime-config-connection.html
.. _use an SSH tunnel:
    http://www.postgresql.org/docs/9.3/static/ssh-tunnels.html
.. _WSGI: http://wsgi.readthedocs.org/
.. _POSBox: https://flectrahq.com/point-of-sale
