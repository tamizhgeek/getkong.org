---
title: Configuration Reference
---

# Configuration Reference

Kong's configuration file is a [YAML][yaml] file that can be specified when using Kong through the [CLI][cli-reference]. This file allows you
to configure and customize Kong to your needs. From the ports it uses, the database it conncts to, and even the internal NGINX server itself.

## Where should I place my configuration file?

When using Kong, you can specify the location of your configuration file from any command using the `-c` argument. See the [CLI reference][cli-reference] for more information.

However, when no configuration file is passed to Kong, it will look under `/etc/kong/kong.yml` for a fallback configuration file. Should no file be present in this location, Kong will then load a default configuration from its Luarocks install path.

## Property Reference

This reference describes every property defined in a typical configuration file and their default values.

They are all **required**.

### Summary

- [**proxy_port**](#proxy_port)
- [**admin_api_port**](#admin_api_port)
- [**nginx_working_dir**](#nginx_working_dir)
- [**plugins_available**](#plugins_available)
- [**send_anonymous_reports**](#send_anonymous_reports)
- [**databases_available**](#databases_available.*)
- [**database**](#database)
- [**database_cache_expiration**](#database_cache_expiration)
- [**nginx**](#nginx)

---

### `proxy_port`

Port which Kong proxies requests through, developers using your API will make requests against this port.

**Default:**

```yaml
proxy_port: 8000
```

---

### `admin_api_port`

Port which the [RESTful Admin API](/docs/{{page.kong_version}}/admin-api/) is served through.

**Note:** This port is used to manage your Kong instances, therefore it should be placed behind a firewall
or closed off network to ensure security.

**Default:**

```yaml
admin_api_port: 8001
```

---

### `nginx_working_dir`

Similar to the NGINX `--prefix` option, it defines a directory that will contain server files, such as access and error logs, or the Kong pid file.

**Default:**

```yaml
nginx_working_dir: /usr/local/kong/
```

---

### `plugins_available`



A list of plugins installed on this node that Kong will load and try to execute during the lifetime of a request. Kong will look for a [`plugin configuration`](/docs/{{page.kong_version}}/admin-api/#plugin-object) entry for each plugin in this list during each request to determine whether the plugin should be executed. Removing plugins from this list will reduce load on your Kong instance.

**Default:**

```yaml
plugins_available:
  - keyauth
  - basicauth
  - ratelimiting
  - tcplog
  - udplog
  - filelog
  - request_transformer
  - cors
```

---

### `send_anonymous_reports`

If set to `true`, Kong will send anonymous error reports to Mashape. This helps Mashape maintaining and improving Kong.

**Default:**

```yaml
send_anonymous_reports: true
```

---

### `databases_available.*`

A dictionary of databases Kong can connect to, and their respective properties.

Currently, Kong only supports [Cassandra v{{site.data.kong_latest.dependencies.cassandra}}](http://cassandra.apache.org/) as a database.

  **`databases_available.*.properties`**

  A dictionary of properties needed for Kong to connect to a given database (where `.*` is the name of the database).

**Default:**

```yaml
databases_available:
  cassandra:
    properties:
      hosts: "localhost"
      port: 9042
      timeout: 1000
      keyspace: kong
      keepalive: 60000
```

---

### `database`

The desired database to use for this Kong instance as a string, matching one of the databases defined under [`databases_available`](#databases_available).

**Default:**

```yaml
database: cassandra
```

---

### `database_cache_expiration`

A value specifying (in seconds) how long Kong will keep database entities in memory. Setting this to a high value will cause Kong to avoid making multiple queries to the database in order to retrieve an API's target URL. However, this also means you may be required to wait a while before the
cached value is flushed and reflects any potential changes made during that time.

**Default:**

```yaml
database_cache_expiration: 5 # in seconds
```

---

### `nginx`

The NGINX configuration (or `nginx.conf`) that will be used for this instance.

**Warning:** Modifying the NGINX configuration can lead to unexpected results, edit the configuration only if you are confident about doing so.

**Default:**

```yaml
nginx: |
  worker_processes auto;
  error_log logs/error.log error;
  daemon on;

  worker_rlimit_nofile {{auto_worker_rlimit_nofile}};

  env KONG_CONF;

  events {
    worker_connections {{auto_worker_connections}};
    multi_accept on;
  }

  http {
    resolver {{dns_resolver}} ipv6=off;
    charset UTF-8;

    access_log logs/access.log;
    access_log off;

    # Timeouts
    keepalive_timeout 60s;
    client_header_timeout 60s;
    client_body_timeout 60s;
    send_timeout 60s;

    # Proxy Settings
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    proxy_ssl_server_name on;

    # IP Address
    real_ip_header X-Forwarded-For;
    set_real_ip_from 0.0.0.0/0;
    real_ip_recursive on;

    # Other Settings
    client_max_body_size 0;
    underscores_in_headers on;
    reset_timedout_connection on;
    tcp_nopush on;

    ################################################
    #  The following code is required to run Kong  #
    # Please be careful if you'd like to change it #
    ################################################

    # Lua Settings
    lua_package_path ';;';
    lua_code_cache on;
    lua_max_running_timers 4096;
    lua_max_pending_timers 16384;
    lua_shared_dict locks 100k;
    lua_shared_dict cache {{memory_cache_size}}m;
    lua_socket_log_errors off;

    init_by_lua '
      kong = require "kong"
      local status, err = pcall(kong.init)
      if not status then
        ngx.log(ngx.ERR, "Startup error: "..err)
        os.exit(1)
      end
    ';

    init_worker_by_lua 'kong.exec_plugins_init_worker()';

    server {
      server_name _;
      listen {{proxy_port}};
      listen {{proxy_ssl_port}} ssl;

      ssl_certificate_by_lua 'kong.exec_plugins_certificate()';

      ssl_certificate {{ssl_cert}};
      ssl_certificate_key {{ssl_key}};

      location / {
        default_type 'text/plain';

        # This property will be used later by proxy_pass
        set $backend_url nil;

        # Authenticate the user and load the API info
        access_by_lua 'kong.exec_plugins_access()';

        # Proxy the request
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass $backend_url;
        proxy_pass_header Server;

        # Add additional response headers
        header_filter_by_lua 'kong.exec_plugins_header_filter()';

        # Change the response body
        body_filter_by_lua 'kong.exec_plugins_body_filter()';

        # Log the request
        log_by_lua 'kong.exec_plugins_log()';
      }

      location /robots.txt {
        return 200 'User-agent: *\nDisallow: /';
      }

      error_page 500 /500.html;
      location = /500.html {
        internal;
        content_by_lua '
          local responses = require "kong.tools.responses"
          responses.send_HTTP_INTERNAL_SERVER_ERROR("An unexpected error occurred")
        ';
      }
    }

    server {
      listen {{admin_api_port}};

      location / {
        default_type application/json;
        content_by_lua '
          ngx.header["Access-Control-Allow-Origin"] = "*"
          if ngx.req.get_method() == "OPTIONS" then
            ngx.header["Access-Control-Allow-Methods"] = "GET,HEAD,PUT,PATCH,POST,DELETE"
            ngx.exit(204)
          end
          local lapis = require "lapis"
          lapis.serve("kong.api.app")
        ';
      }

      location /nginx_status {
        internal;
        stub_status;
      }

      location /robots.txt {
        return 200 'User-agent: *\nDisallow: /';
      }

      # Do not remove, additional configuration placeholder for some plugins
      # {{additional_configuration}}
    }
  }
```

[cli-reference]: /docs/{{page.kong_version}}/cli
[yaml]: http://yaml.org
