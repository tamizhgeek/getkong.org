---
sitemap: true
id: page-plugin
title: Plugins - HTTP Log
header_title: HTTP Log
header_icon: /assets/images/icons/plugins/http-log.png
header_btn_text: Report Bug
header_btn_href: https://github.com/Mashape/kong/issues/new
header_btn_target: _blank
breadcrumbs:
  Plugins: /plugins
---

Send request and response logs to an HTTP server.

---

## Installation

Add the plugin to the list of available plugins on every Kong server in your cluster by editing the [kong.yml][configuration] configuration file:

```yaml
plugins_available:
  - httplog
```

Every node in the Kong cluster should have the same `plugins_available` property value.

## Configuration

Configuring the plugin is straightforward, you can add it on top of an [API][api-object] (or [Consumer][consumer-object]) by executing the following request on your Kong server:

```bash
$ curl -X POST http://kong:8001/apis/{api}/plugins \
    --data "name=httplog" \
    --data "value.http_endpoint=http://mockbin.org/bin/:id/" \
    --data "value.method=POST" \
    --data "value.timeout=1000" \
    --data "value.keepalive=1000"
```

`api`: The `id` or `name` of the API that this plugin configuration will target

form parameter                          | description
 ---                                    | ---
`name`                                  | The name of the plugin to use, in this case: `httplog`
`consumer_id`<br>*optional*             | The CONSUMER ID that this plugin configuration will target. This value can only be used if [authentication has been enabled][faq-authentication] so that the system can identify the user making the request.
`value.http_endpoint`                   | The HTTP endpoint (including the protocol to use) where to send the data to
`value.method`                          | Default `POST`. An optional method used to send data to the http server, other supported values are PUT, PATCH
`value.timeout`                         | Default `10000`. An optional timeout in milliseconds when sending data to the upstream server
`value.keepalive`                       | Default `60000`. An optional value in milliseconds that defines for how long an idle connection will live before being closed

[api-object]: /docs/{{site.data.kong_latest.version}}/admin-api/#api-object
[configuration]: /docs/{{site.data.kong_latest.version}}/configuration
[consumer-object]: /docs/{{site.data.kong_latest.version}}/admin-api/#consumer-object
[faq-authentication]: /docs/{{site.data.kong_latest.version}}/faq/#how-can-i-add-an-authentication-layer-on-a-microservice/api?

## Log Format

Every request will be logged separately in a JSON object, with the following format:

```json
{
    "request": {
        "method": "GET",
        "uri": "/get",
        "size": "75",
        "request_uri": "http://httpbin.org:8000/get",
        "querystring": {},
        "headers": {
            "accept": "*/*",
            "host": "httpbin.org",
            "user-agent": "curl/7.37.1"
        }
    },
    "response": {
        "status": 200,
        "size": "434",
        "headers": {
            "Content-Length": "197",
            "via": "kong/0.3.0",
            "Connection": "close",
            "access-control-allow-credentials": "true",
            "Content-Type": "application/json",
            "server": "nginx",
            "access-control-allow-origin": "*"
        }
    },
    "api": {
        "public_dns": "test.com",
        "target_url": "http://httpbin.org/",
        "created_at": 1432855823000,
        "name": "test.com",
        "id": "fbaf95a1-cd04-4bf6-cb73-6cb3285fef58"
    },
    "latencies": {
        "proxy": 1430,
        "kong": 9,
        "request": 1921
    },
    "started_at": 1433209822425,
    "client_ip": "127.0.0.1"
}
```

A few considerations on the above JSON object:

* `request` contains properties about the request sent by the client
* `response` contains properties about the response sent to the client
* `api` contains Kong properties about the specific API requested
* `latencies` contains some data about the latencies involved:
   * `proxy` is the time it took for the final service to process the request
   * `kong` is the internal Kong latency that it took to run all the plugins
   * `request` is the time elapsed between the first bytes were read from the client and after the last bytes were sent to the client. Useful for detecting slow clients.
