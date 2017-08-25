Kong Heroku app
===============
Deploy [Kong CE 0.11.0](http://blog.mashape.com/kong-ce-0-11-0-released/) clusters to Heroku Common Runtime and Private Spaces.

Uses the [Kong buildpack](https://github.com/heroku/heroku-buildpack-kong).

Requirements
------------
* [Heroku](https://www.heroku.com/home)
  * [command-line tools (CLI)](https://toolbelt.heroku.com)
  * [a free account](https://signup.heroku.com)
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Usage
-----
Get started by cloning heroku-kong and deploying it to a new Heroku app.

```bash
git clone https://github.com/heroku/heroku-kong.git
cd heroku-kong

# Create app in Common Runtime:
heroku create my-proxy-app --buildpack https://github.com/heroku/heroku-buildpack-kong.git
# …or in a Private Space:
heroku create my-proxy-app --buildpack https://github.com/heroku/heroku-buildpack-kong.git --space my-private-space

heroku addons:create heroku-postgresql:hobby-dev

git push heroku master
# …the first build will take approximately ten minutes; subsequent builds approx two-minutes.
```

### Admin Console

Use Kong CLI and the Admin API in a [one-off dyno](https://devcenter.heroku.com/articles/one-off-dynos):

```bash
$ heroku run bash

# Run Kong in the background:
~ $ bin/background-start

# Then, use `curl` to issue Admin API commands
# and `jq` to format the output:
~ $ curl http://$KONG_ADMIN_LISTEN | jq

# Example CLI commands:
# (note some commands require the config file and others the prefix)
~ $ kong migrations list -c $KONG_CONF
~ $ kong health -p /app/.heroku
~ $ kong stop -p /app/.heroku
```

### Configuration

The Heroku app must have several [config vars, as defined in the buildpack](https://github.com/heroku/heroku-buildpack-kong#usage).

Kong is automatically configured at runtime with the `.profile.d/kong-12f.sh` script, which:

  * renders the `config/kong.conf` file
  * exports environment variables (see: `.profile.d/kong-env` in a running dyno)
  * may all be overridden by setting `KONG_`-prefixed config vars, e.g. `heroku config:set KONG_LOG_LEVEL=debug`

Revise [`config/kong.conf.etlua`](config/kong.conf.etlua) to suite your application.

See: [Kong 0.11 Configuration Reference](https://getkong.org/docs/0.11.x/configuration/)

### Kong plugins & additional Lua modules

See [buildpack usage](https://github.com/heroku/heroku-buildpack-kong#usage)

### Protecting the Admin API
Kong's Admin API has no built-in authentication. Its exposure must be limited to a restricted, private network.

For Kong on Heroku, the Admin API listens privately at the value of environment variable `KONG_ADMIN_LISTEN` or `admin_listen` in [`config/kong.conf.etlua`](config/kong.conf.etlua), which defaults to port `8001`.

#### Authenticated Admin API
Using Kong itself, you may expose the Admin API with authentication & rate limiting.

From the [admin console](#user-content-admin-console):
```bash
# Create the authenticated `/kong-admin` API, targeting the localhost port:
curl http://localhost:8001/apis -i -X POST \
  --data name=kong-admin
  --data uris=/kong-admin
  --data upstream_url=http://localhost:8001 | jq
curl -i -X POST --url http://localhost:8001/apis/kong-admin/plugins/ --data 'name=request-size-limiting' --data "config.allowed_payload_size=8"
curl -i -X POST --url http://localhost:8001/apis/kong-admin/plugins/ --data 'name=rate-limiting' --data "config.minute=12"
curl -i -X POST --url http://localhost:8001/apis/kong-admin/plugins/ --data 'name=key-auth' --data "config.hide_credentials=true"
curl -i -X POST --url http://localhost:8001/apis/kong-admin/plugins/ --data 'name=acl' --data "config.whitelist=kong-admin"

# Create a consumer with username and authentication credentials:
curl -i -X POST --url http://localhost:8001/consumers/ --data 'username=8th-wonder'
curl -i -X POST --url http://localhost:8001/consumers/8th-wonder/acls --data 'group=kong-admin'
curl -i -X POST --url http://localhost:8001/consumers/8th-wonder/key-auth
# …this response contains the `"key"`.
```

Now, access Kong's Admin API via the protected, public-facing proxy:
```bash
# Set the request header:
curl -H 'apikey: {kong-admin key}' https://kong-proxy-public.herokuapp.com/kong-admin/status
# or use query params:
curl https://kong-proxy-public.herokuapp.com/kong-admin/status?apikey={kong-admin key}
```


### Demo: [API Rate Limiting](https://getkong.org/plugins/rate-limiting/)

Request [this Bay Lights API](https://kong-proxy-public.herokuapp.com/bay-lights/lights) more than five times in a minute, and you'll get **HTTP Status 429: API rate limit exceeded**, along with `X-Ratelimit-Limit-Minute` & `X-Ratelimit-Remaining-Minute` headers to help the API consumers regulate their usage.

Try it in your shell terminal:
```bash
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 200 OK
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 200 OK
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 200 OK
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 200 OK
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 200 OK
curl -I https://kong-proxy-public.herokuapp.com/bay-lights/lights
# HTTP/1.1 429
```

Here's the whole configuration for this API rate limiter:

```bash
curl -i -X POST --url http://localhost:8001/apis/ --data 'name=bay-lights' --data 'upstream_url=https://bay-lights-api-production.herokuapp.com/' --data 'request_path=/bay-lights' --data 'strip_request_path=true'
curl -i -X POST --url http://localhost:8001/apis/bay-lights/plugins/ --data 'name=request-size-limiting' --data "config.allowed_payload_size=8"
curl -i -X POST --url http://localhost:8001/apis/bay-lights/plugins/ --data 'name=rate-limiting' --data "config.minute=5"
# Demo loading app-specific Kong plugins & Lua modules.
curl -i -X POST --url http://localhost:8001/apis/bay-lights/plugins/ --data 'name=hello-world-header'
```

### Demo: API translation, XML as JSON

JSON/REST has taken over as the internet API lingua franca, shedding the complexity of XML/SOAP. The [National Digital Forecast Database [NDFD]](http://graphical.weather.gov/xml/) is a legacy XML/SOAP service.

Here we demonstrate a custom plugin [ndfd-xml-as-json](lib/kong/plugins/ndfd-xml-as-json) to expose an JSON/REST API that fetches the maximum temperatures forecast for a location from the NDFD SOAP service. Using the single-resource concept of REST, the many variations of a SOAP interface may be broken out into elegant, individual JSON APIs.

Try it in your shell terminal:
```bash
curl --data '{"latitude":37.733795,"longitude":-122.446747}' https://kong-proxy-public.herokuapp.com/ndfd-max-temps
# Response contains max temperatures forecast for San Francisco, CA
curl --data '{"latitude":27.964157,"longitude":-82.452606}' https://kong-proxy-public.herokuapp.com/ndfd-max-temps
# Response contains max temperatures forecast for Tampa, FL
curl --data '{"latitude":41.696629,"longitude":-71.149994}' https://kong-proxy-public.herokuapp.com/ndfd-max-temps
# Response contains max temperatures forecast for Fall River, MA
```

Much more elegant than the legacy API. See the [sample request body](spec/data/ndfd-request.xml):
```bash
curl --data @spec/data/ndfd-request.xml -H 'Content-Type:text/xml' -X POST http://graphical.weather.gov/xml/SOAP_server/ndfdXMLserver.php
# Response contains wrapped XML data. Enjoy decoding that.
```

This technique may be used to create a suite of cohesive JSON APIs out of various legacy APIs.

Here's the configuration for this API translator:

```bash
curl -X POST -v http://localhost:8001/apis --data 'name=ndfd-max-temps' --data 'upstream_url=http://graphical.weather.gov/xml/SOAP_server/ndfdXMLserver.php' --data 'request_path=/ndfd-max-temps' --data 'strip_request_path=true'
curl -X POST -v http://localhost:8001/apis/ndfd-max-temps/plugins/ --data 'name=request-size-limiting' --data "config.allowed_payload_size=8"
curl -X POST -v http://localhost:8001/apis/ndfd-max-temps/plugins/ --data 'name=rate-limiting' --data "config.minute=5"
curl -X POST -v http://localhost:8001/apis/ndfd-max-temps/plugins/ --data 'name=ndfd-xml-as-json'
```

### Demo: API analytics, [Librato](https://elements.heroku.com/addons/librato)

Collect per-API metrics, explore, and set alerts on them with Librato. This [`librato-analytics`](lib/kong/plugins/librato-analytics) plugin demonstrates near-realtime (~1-minute delay), batch-oriented (up 300 metrics/post), asynchronous (non-blocking to proxy traffic) pushes of Kong/Nginx metrics to [Librato's Metrics API](http://dev.librato.com/v1/metrics).

![Screenshot of Librato Kong metrics](http://marsikai.s3.amazonaws.com/librato-kong-bay-lights.png)

The per-API metrics are sent by source, named "kong-{API-NAME}":
  * request size (bytes)
  * response size (bytes)
  * kong latency (milliseconds)
  * upstream latency (milliseconds)
  * response latency (milliseconds)

*This demo requires your own Heroku Kong instance with the Librato add-on. Kong sends custom metrics, so a paid plan of any level is required.*

Here's the plugin configuration. Example based on the Bay Lights API example above:

```bash
curl -X POST -v http://localhost:8001/apis/bay-lights/plugins/ --data 'name=librato-analytics' --data "config.verify_ssl=false"
```

The `LIBRATO_*` config vars set-up by the add-on will be used for authorization, but can be overridden by explicitly setting `config.username` & `config.token` for specific instances of the Kong plugin.

### Dev Notes

#### Learning the Language of Kong

* [Definitely an openresty guide](http://www.staticshin.com/programming/definitely-an-open-resty-guide/)
* [An Introduction To OpenResty - Part 1](http://openmymind.net/An-Introduction-To-OpenResty-Nginx-Lua/), [2](http://openmymind.net/An-Introduction-To-OpenResty-Part-2/), & [3](http://openmymind.net/An-Introduction-To-OpenResty-Part-3/)
* [Nginx API for Lua](https://github.com/openresty/lua-nginx-module#nginx-api-for-lua), `ngx` reference, for use in Kong plugins
  * [Nginx variables](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_status), accessible through `ngx.var`

#### Programming with Lua

* [Lua 5.1](http://www.lua.org/manual/5.1/), Note: Kong is not compatible with the newest Lua version
* [Classic Objects](https://github.com/rxi/classic), the basis of Kong's plugins
* [Moses](http://yonaba.github.io/Moses/doc/), functional programming
* [Lubyk](http://doc.lubyk.org), realtime programming (performance- & game-oriented)
* [resty-http](https://github.com/pintsized/lua-resty-http), Nginx-Lua co-routine based HTTP client
* [Serpent](http://notebook.kulchenko.com/programming/serpent-lua-serializer-pretty-printer), inspect values
* [Busted](http://olivinelabs.com/busted/), testing framework

#### Using Environment Variables in Plugins

As a [12-factor](http://12factor.net) app, Heroku Kong already uses environment variables for configuration. Here's how to use those vars within your own code.

1. Whitelist the variable name for use within Nginx 
  * In a [custom nginx config](https://getkong.org/docs/0.11.x/configuration/#custom-nginx-configuration) add `env MY_VARIABLE;`
2. Access the variable in Lua plugins
  * Use `os.getenv('MY_VARIABLE')` to retrieve the value

#### Local Development

To work with Kong locally on Mac OS X.

##### Setup

1. [Install Kong using the .pkg](https://getkong.org/install/osx/)
1. Execute `./bin/setup`

##### Running

* Execute `./bin/start`
* Logs in `/usr/local/var/kong/logs/` 

##### Testing

Any test-specific Lua rocks should be specified in `.luarocks_test` file, so that they are not installed when the app is deployed.

1. Add tests in `spec/`
  * Uses the [Busted testing framework](http://olivinelabs.com/busted)
  * See also [Kong integration testing](https://getkong.org/docs/0.5.x/plugin-development/tests/)
1. Execute `source .profile.local`
1. Execute `busted` to run the tests

