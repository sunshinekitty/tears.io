---
layout: post
comments: true
title: Enabling SSL on Horizon dashboard in Openstack Mitaka
preview: I recently went looking around to see if there was any special configuration needed to enable SSL on the Horizon dashboard.  I found lots of articles saying you need to modify `local_settings.py`.  It looks like in Mitaka this is no longer the case.
---

I recently went looking around to see if there was any special configuration needed to enable SSL on the Horizon dashboard.  I found lots of articles saying you need to modify `local_settings.py`.  It looks like in Mitaka this is no longer the case.

You should just need to configure your Apache vhost for Horizon.  I have placed an example below. Note that in this example I force SSL.  You can also allow both HTTP and HTTPS by removing the redirect and duplicating the configuration into the :80 vhost.

```
<VirtualHost *:80>
  ServerName horizon.tears.io

  ServerAlias 192.168.0.100
  ServerAlias horizon.tears.io
  ServerAlias localhost

  ## Force redirect to SSL website
  RewriteEngine On
  RewriteCond %{HTTPS} !on
  RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

<VirtualHost *:443>
  ServerName horizon.tears.io

  ## Vhost docroot
  DocumentRoot "/var/www/"
  ## Alias declarations for resources outside the DocumentRoot
  Alias /dashboard/static "/usr/share/openstack-dashboard/static"

  ## Directories, there should at least be a declaration for /var/www/

  <Directory "/var/www/">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  ## Logging
  ErrorLog "/var/log/httpd/horizon_error.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/horizon_access.log" combined

  ## RedirectMatch rules
  RedirectMatch permanent  ^/$ /dashboard

  ## Server aliases
  ServerAlias 192.168.0.100
  ServerAlias horizon.tears.io
  ServerAlias localhost
  WSGIDaemonProcess dashboard group=apache processes=3 threads=10 user=apache
  WSGIProcessGroup dashboard
  WSGIScriptAlias /dashboard "/usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi"

  ## SSL Related, replace paths with your own
  SSLEngine on
  SSLCertificateFile    /etc/pki/tls/certs/tears.crt
  SSLCertificateKeyFile /etc/pki/tls/private/tears.key
</VirtualHost>
```
