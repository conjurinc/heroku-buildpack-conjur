# Heroku Buildpack: Conjur

Conjur-buildpack provides a Conjur access-controlled gateway to a Heroku app.

To that end, it bases on https://github.com/ryandotsmith/nginx-buildpack to
vendor a nginx reverse proxy which checks permissions before allowing requests
to go through.

## Versions

* Buildpack version: 0.1.0
* Base NGINX buildpack Version: 0.4
* NGINX Version: 1.7.12

## Requirements

* Your webserver listens to the socket at `/tmp/nginx.socket`.
* You touch `/tmp/app-initialized` when you are ready for traffic.
* You can start your web server with a shell command.
* You have a Conjur resource representing this service.

## Features

* Uses Authorization header on incoming request to check permission with Conjur.
* Distinguishes between GET and other requests.
* SSL certificate verification.
* Authorization result caching.

### Delegate authorization

The reverse proxy passes the Authorization header of the incoming request to
Conjur authz to perform privilege check on the resource associated with this
service. It denies the request if the check is negative.

### HTTP method mapping

For GET requests to succeed the client must have *read* privilege; for all the
other HTTP methods *update* privilege is checked.

### SSL certificate verification

By default a leap-of-faith SSL verification is performed -- on starting up, the
Conjur server is contacted and its root certificate is stored in a file, which
is then used for subsequent verification. If you want to make it safer, you
can set `CONJUR_CERT_FINGERPRINT` Heroku variable to the expected root
certificate fingerprint (in format `92:25:4F:70:...`, as output by
`openssl x509 -fingerprint`); the server won't start if it doesn't match.

### Authorization result caching

Successful authorizations are automatically cached on per-token basis. This
means this gateway is suitable even for interactive applications with plenty of
resources (and therefore, requests), as only a single authorization request
will be performed for the entire lifetime of a token.

### Language/App Server Agnostic

Conjur-buildpack provides a command named `bin/start-nginx` this command takes another command as an argument. You must pass your app server's startup command to `start-nginx`.

For example, to get NGINX and Unicorn up and running:

```bash
$ cat Procfile
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```

### Application/Dyno coordination

The buildpack will not start NGINX until a file has been written to `/tmp/app-initialized`. Since NGINX binds to the dyno's $PORT and since the $PORT determines if the app can receive traffic, you can delay NGINX accepting traffic until your application is ready to handle it. The examples below show how/when you should write the file when working with Unicorn.

## Setup

Here are 2 setup examples. One example for a new app, another for an existing app. In both cases, we are working with ruby & unicorn. Keep in mind that this buildpack is not ruby specific.

### Existing App

Update Buildpacks
```bash
$ heroku buildpack:set https://github.com/ddollar/heroku-buildpack-multi.git
$ echo 'https://github.com/conjurinc/conjur-buildpack.git' >> .buildpacks
$ echo 'https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby.tgz' >> .buildpacks
$ git add .buildpacks
$ git commit -m 'Add multi-buildpack'
```
Update Procfile:
```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```
```bash
$ git add Procfile
$ git commit -m 'Update procfile for Conjur buildpack'
```
Update Unicorn Config
```ruby
require 'fileutils'
listen '/tmp/nginx.socket'
before_fork do |server,worker|
	FileUtils.touch('/tmp/app-initialized')
end
```
```bash
$ git add config/unicorn.rb
$ git commit -m 'Update unicorn config to listen on NGINX socket.'
```

Create a Conjur resource
```sh-session
$ conjur resource create service:unicorn
$ heroku config:set CONJUR_RESOURCE_URL=https://conjur.example.com/api/authz/account/resources/service/unicorn
```

Deploy Changes
```bash
$ git push heroku master
```

### New App

```sh-session
$ mkdir myapp; cd myapp
$ git init
```

**Gemfile**
```ruby
source 'https://rubygems.org'
gem 'unicorn'
```

**config.ru**
```ruby
run Proc.new {[200,{'Content-Type' => 'text/plain'}, ["hello world"]]}
```

**config/unicorn.rb**
```ruby
require 'fileutils'
preload_app true
timeout 5
worker_processes 4
listen '/tmp/nginx.socket', backlog: 1024

before_fork do |server,worker|
	FileUtils.touch('/tmp/app-initialized')
end
```
Install Gems
```sh-session
$ bundle install
```
Create Procfile
```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```
Create a Conjur resource
```sh-session
$ conjur resource create service:unicorn
```
Create & Push Heroku App:
```sh-session
$ heroku create --buildpack https://github.com/ddollar/heroku-buildpack-multi.git
$ heroku config:set CONJUR_RESOURCE_URL=https://conjur.example.com/api/authz/account/resources/service/unicorn
$ echo 'https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby.tgz' >> .buildpacks
$ echo 'https://github.com/conjurinc/conjur-buildpack.git' >> .buildpacks
$ git add .
$ git commit -am "init"
$ git push heroku master
$ heroku logs -t
```
Visit App
```sh-session
$ conjur proxy https://`heroku domains | tail -n+2` &
$ xdg-open http://localhost:8080
$ heroku open # => will 401 -- no Conjur authorization header
```

## License
Copyright (c) 2015 Rafal Rzepecki
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### Original NGINX buildpack license
Copyright (c) 2013 Ryan R. Smith
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
