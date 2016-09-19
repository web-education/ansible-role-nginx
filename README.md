nginx
=====

This role installs and configures the nginx web server. The user can specify
any http configuration parameters they wish to apply their site. Any number of
sites can be added with configurations of your choice.
It also has a special template to configure nginx for [ENTCore](https://github.com/entcore/entcore)

[![Build Status](https://travis-ci.org/jdauphant/ansible-role-nginx.svg?branch=master)](https://travis-ci.org/jdauphant/ansible-role-nginx)

Requirements
------------

This role requires Ansible 1.4 or higher and platform requirements are listed
in the metadata file.  
For FreeBSD a working pkgng setup is required (see: https://www.freebsd.org/doc/handbook/pkgng-intro.html )

Role Variables
--------------

The variables that can be passed to this role and a brief description about
them are as follows. (For all variables, take a look at [defaults/main.yml](defaults/main.yml))

```yaml
# The user to run nginx
nginx_user: "www-data"

# A list of directives for the events section.
nginx_events_params:
 - worker_connections 512
 - debug_connection 127.0.0.1
 - use epoll
 - multi_accept on

# A list of hashs that define the servers for nginx,
# as with http parameters. Any valid server parameters
# can be defined here.
nginx_sites:
 default:
     - listen 80
     - server_name _
     - root "/usr/share/nginx/html"
     - index index.html
 foo:
     - listen 8080
     - server_name localhost
     - root "/tmp/site1"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ { try_files $uri $uri/ /index.html; }
 bar:
     - listen 9090
     - server_name ansible
     - root "/tmp/site2"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ {
         try_files $uri $uri/ /index.html;
         allow 127.0.0.1;
         deny all;
       }

# A list of hashs that define additional configuration
nginx_configs:
  proxy:
      - proxy_set_header X-Real-IP  $remote_addr
      - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
  upstream:
      - upstream foo { server 127.0.0.1:8080 weight=10; }
  geo:
      - geo $local {
          default 0;
          127.0.0.1 1;
        }
  gzip:
      - gzip on
      - gzip_disable msie6

# A list of hashs that define user/password files
nginx_auth_basic_files:
   demo:
     - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
     - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo

```

Examples
========

1) Install nginx with HTTP directives of choices, but with no sites
configured and no additionnal configuration:

```yaml
- hosts: all
  roles:
  - {role: nginx,
     nginx_http_params: ["sendfile on", "access_log /var/log/nginx/access.log"]
                          }
```

2) Install nginx with different HTTP directives than previous example, but no
sites configured and no additionnal configuration.

```yaml
- hosts: all
  roles:
  - {role: nginx,
     nginx_http_params: ["tcp_nodelay on", "error_log /var/log/nginx/error.log"]}
```

Note: Please make sure the HTTP directives passed are valid, as this role
won't check for the validity of the directives. See the nginx documentation
for details.

3) Install nginx and add a site to the configuration.

```yaml
- hosts: all

  roles:
  - role: nginx
    nginx_http_params:
      - sendfile "on"
      - access_log "/var/log/nginx/access.log"
    nginx_sites:
      bar:
        - listen 8080
        - location / { try_files $uri $uri/ /index.html; }
        - location /images/ { try_files $uri $uri/ /index.html; }
    nginx_configs:
      proxy:
        - proxy_set_header X-Real-IP  $remote_addr
        - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
```

Note: Each site added is represented by list of hashes, and the configurations
generated are populated in /etc/nginx/site-available/, a link is from /etc/nginx/site-enable/ to /etc/nginx/site-available

The file name for the specific site configurtaion is specified in the hash
with the key "file_name", any valid server directives can be added to hash.
Additional configuration are created in /etc/nginx/conf.d/

4) Install Nginx , add 2 sites (different method) and add additional configuration

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
         foo:
           - listen 8080
           - server_name localhost
           - root /tmp/site1
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
         bar:
           - listen 9090
           - server_name ansible
           - root /tmp/site2
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
      nginx_configs:
         proxy:
            - proxy_set_header X-Real-IP  $remote_addr
            - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
```

5) Install Nginx , add 2 sites, add additional configuration and an upstream configuration block

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_error_log_level: info
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
        foo:
           - listen 8080
           - server_name localhost
           - root /tmp/site1
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
        bar:
           - listen 9090
           - server_name ansible
           - root /tmp/site2
           - if ( $host = example.com ) { rewrite ^(.*)$ http://www.example.com$1 permanent; }
           - location / {
             try_files $uri $uri/ /index.html;
             auth_basic            "Restricted";
             auth_basic_user_file  auth_basic/demo;
           }
           - location /images/ { try_files $uri $uri/ /index.html; }
      nginx_configs:
        proxy:
            - proxy_set_header X-Real-IP  $remote_addr
            - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
        upstream:
            # Results in:
            # upstream foo_backend {
            #   server 127.0.0.1:8080 weight=10;
            # }
            - upstream foo_backend { server 127.0.0.1:8080 weight=10; }
      nginx_auth_basic_files:
        demo:
           - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
           - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo
```

6) Install Nginx, add a site and use special yaml syntax to make the location blocks multiline for clarity

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
        foo:
           - listen 443 ssl
           - server_name foo.example.com
           - set $myhost foo.example.com
           - |
             location / {
               proxy_set_header Host foo.example.com;
             }
           - |
             location ~ /v2/users/.+?/organizations {
               if ($request_method = PUT) {
                 set $myhost bar.example.com;
               }
               if ($request_method = DELETE) {
                 set $myhost bar.example.com;
               }
               proxy_set_header Host $myhost;
             }
```
7) Example to use this role with my ssl-certs role to generate or copie ssl certificate ( https://galaxy.ansible.com/list#/roles/3115 )
```yaml
 - hosts: all
   roles:
     - jdauphant.ssl-certs
     - role: jdauphant.nginx
       nginx_configs:
          ssl:
               - ssl_certificate_key {{ssl_certs_privkey_path}}
               - ssl_certificate     {{ssl_certs_cert_path}}
       nginx_sites:
          default:
               - listen 443 ssl
               - server_name _
               - root "/usr/share/nginx/html"
               - index index.html
```
8) Example vars file for [ENTCore](https://github.com/entcore/entcore) setup:
```yaml
# The user to run nginx
nginx_user: "www-data"
nginx_worker_processes: 'auto'
nginx_worker_rlimit_nofile: 100480
# A list of directives for the events section.
nginx_events_params:
 - worker_connections 8192
 - use epoll
 - multi_accept on

# A list of hashs that define the servers for nginx,
# as with http parameters. Any valid server parameters
# can be defined here.
nginx_sites:
 server-status:
     - listen 80
     - server_name localhost
     - location /nginx_status {
         stub_status on;
         access_log off;
         allow 127.0.0.1;
         deny all;
       }
ent_sites:
 cool-ent:
   server_name: 'cool-ent.opendigitaleducation.com'
   nodes:
     - '10.0.0.1'
     - '10.0.0.2'
   ssl_certificate: '/etc/ssl/star.opendigitaleducation.com/crt'
   ssl_certificate_key: '/etc/ssl/star.opendigitaleducation.com/key'
   upstream_suffix: 'vertx.lan'
   serve_statics: true
   statics_path: '/var/www/web-education/static'

nginx_configs:
  proxy:
      - proxy_set_header X-Real-IP  $remote_addr
      - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
  gzip:
      - gzip on
      - gzip_disable msie6

```
For ENTCore's applications' upstreams, a list of web services must be defined:
```yaml
web_services:
  - name: 'portal'
    port: 8017
  - name: 'infra'
    port: 8001
  - name: 'directory'
    port: 8003
  - name: 'history'
    port: 8006
  - name: 'auth'
    port: 8009
  - name: 'workspace'
    port: 8011
  - name: 'appregistry'
    port: 8012
  - name: 'communication'
    port: 8015
  - name: 'workspace'
    port: 8011
    options:
      - 'client_max_body_size 1024M'
```

Dependencies
------------

None

License
-------
BSD

Author Information
------------------

- Original : Benno Joy
- Modified by : DAUPHANT Julien
