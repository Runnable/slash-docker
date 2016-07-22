---
title: Configuration
category: rails
permalink: /rails/docker-configuration
step: 9
tags:
- docker
- rails
- ruby
- configuration
excerpt: Tune your dockerized Rails application.
description: How to properly configure your Ruby on Rails application for Docker deployments. 
---

This article overviews the most important of the Rails configuration tunings that are required for effective dockerization.  

# Architectural concerns
 
The following are significant differences between the Docker environment and traditional servers: 

* Containers, the local filesystem, and the application process **are ephemeral**.
* **No local services are available** because [the application is the only process running in the container](../multiple-processes/).
* **With an image, applications can be run in any environment without additional provisioning.** Configuring environment variables and connecting deployment-specific resources (such as data volumes and [other containers](../linking-containers/)) are the only Docker-friendly ways to parameterize container behavior.  
* **Images should not contain private data** because they may be shared with the development team, the production repositories, and customers. 
 
That said, the principles for configuring dockerized Rails (which correlate with the broader concept of [The Twelve­Factor App](http://12factor.net/)) are fairly easy to derive.


## Configuring dockerized Rails: fundamentals 
* The differences between development, testing, and production configurations should be minimized. 
* All services except the Rails server should be considered external. 
* Secrets and deployment-specific parameters should be provided via environment variables. 
* Files that have been uploaded by users must be pushed to persistent external storage as soon as possible.
* Logs should not be permanently stored on a container filesystem. 
 
Let's walk through various aspects of a Rails configuration and optimize them to ensure that the Rails application works seamlessly in a dockerized environment.

# Database connections
 
Here is an example *database.yml*, which is used in the other articles of this series. By default, the script expects another Docker container (accessible by the hostname "dbserver") to provide passwordless access to the mySQL database "myapp_db." 
  
Environment variables can override any of the options. Configuration is unified across Rails environments to ensure [dev/prod parity](http://12factor.net/dev-prod-parity).

#### config/database.yml
```yaml
default: &default 
    adapter: mysql2 
    encoding: utf8 
    pool: 5 
    timeout: 5000 
    host: <%= ENV['DB_HOST'] || 'dbserver' %>  
    database: <%= ENV['DB_NAME'] || 'myapp_db' %> 
    username: <%= ENV['DB_USER'] || 'root' %> 
    password: <%= ENV['DB_PASSWORD'] || '' %> 
 
development: 
    <<: *default 
 
test: 
    <<: *default 
 
production: 
    <<: *default 
```

# Secrets
 
Avoiding the storage of secrets inside the image is a well-known Docker principle. 
 
The default Rails configuration includes at least one secret, which must be read from an environment variable in production mode. 


#### config/secrets.yml

```yaml
development: 
    secret_key_base: 316248d5...8e5e46971a49 
 
test: 
    secret_key_base: a00528df9c...bea6c5c45 
 
# Do not keep production secrets in the repository, 
# instead read values from the environment. 
production: 
    secret_key_base: <%= ENV["SECRET_KEY_BASE"] %> 
```

Let's modify this file to unify configuration: 

#### config/secrets.yml

```yaml
production: &default 
    secret_key_base: <%= ENV["SECRET_KEY_BASE"] %> 
 
development: 
    <<: *default 
 
test: 
    <<: *default 
```

Your application may include additional custom secrets. It makes sense to ensure that they are always read from environment variables.
 
***Advanced security concerns*** 

*Storing extremely sensitive data in environment variables may not always be acceptable. The appropriate values are exposed to the container's metadata and could be revealed to anyone who has access to the Docker server.* 
 
*For additional protection, consider storing critical data in online storage with advanced access control. Applications can use a one-time authentication token (provided in an environment variable, as described above) to fetch sensitive secrets during initialization.* 

# Mailers
 
If a dockerized application uses mailers, it must rely on an external delivery service.
 
The default delivery method (and a good choice) for [ActionMailer](http://guides.rubyonrails.org/action_mailer_basics.html) is *:smtp*. However, **the default SMTP server location is localhost:25, so connection settings must be explicitly redefined.** 
 
The example configuration reads and applies JSON­-serialized connection settings from the environment variable MAILER_SMTP_SETTINGS. If the variable is absent and the SMTP service is provided by another Docker container, the mailer connects to the *'smtp'* hostname. 
 
Other deployment­-specific parameters are also configurable via environment variables. 

#### config/application.rb

```ruby
# check http://guides.rubyonrails.org/action_mailer_basics.html for detailed SMTP options 
config.action_mailer.smtp_settings = if ENV["MAILER_SMTP_SETTINGS"].present?  
    JSON.load( ENV["MAILER_SMTP_SETTINGS"] 
).symbolize_keys
    else 
        { address: "smtp", port: "1025" } 
    end 
 
config.action_mailer.default_url_options = { host: ENV["MAILER_URLS_HOST"] } 
config.action_mailer.asset_host = ENV["MAILER_ASSET_HOST"] 
 
# enforce SMTP and not be silent about errors 
config.action_mailer.delivery_method = :smtp 
config.action_mailer.raise_delivery_errors = true 
```


The custom setting in *development.rb* is disabled. 

#### config/environments/development.rb

```ruby
#### respect global configuration 
# config.action_mailer.raise_delivery_errors = false 
```


The custom delivery mode in *test.rb* is optional: 

#### config/environments/test.rb

```ruby
# conditionally switch to special delivery mode, which rspecs may rely on 
if ENV["MAILER_TEST_DELIVERY"].present?  
    config.action_mailer.delivery_method = :test 
end 
```


Except for the single optional tuning for *test.rb*, configuration is unified. The application will always try to send emails, even in development and testing modes. 
 
These features exist by design. With Docker, it is easy to plug a mock SMTP container (e.g., one based on the [MailHog](https://hub.docker.com/r/mailhog/mailhog/) image) to provide a production-like experience for troubleshooting and testing without sending anything to the outer world. 

# Workers 
Rails provides a common API ([ActiveJob](http://guides.rubyonrails.org/active_job_basics.html)) to manage background tasks. Satisfying the following conditions will make ActiveJob work reliably with Docker: 
 
1. ActiveJob must be configured to use a persistent processing backend such as 
[Sidekiq](https://github.com/mperham/sidekiq). 
2. The backend must use externalized storage for the message queue. 
3. Hostname­-specific queues must not be used. 
4. Workers must run in separate containers. 
5. Files whose processing will be delayed should be placed in external storage, where they will be available to worker containers. 
 
The suggested configuration uses Sidekiq and reads the Redis URL from an environment variable. If the variable is absent, Sidekiq assumes the Redis server is a Docker container that is accessible by the hostname *redis*. 

#### Gemfile

```
gem 'sidekiq' 
```

#### config/application.rb

```
config.active_job.queue_adapter = :sidekiq 
```

#### config/initializers/sidekiq.rb

```ruby 
redis_url = ENV["SIDEKIQ_REDIS_URL"] || "redis://redis:6379/12" 
 
# initializer includes settings for both worker server and client 
Sidekiq.configure_server do |config| 
    config.redis = { url: redis_url } 
end 
 
Sidekiq.configure_client do |config|
    config.redis = { url: redis_url } 
end 
```

The Redis container can be run from the [official Redis image](https://hub.docker.com/_/redis/). The worker container is launched from the application image with the custom command ``sidekiq`` instead of the default ``rails server``.

# Caching

[Rails caching](http://guides.rubyonrails.org/caching_with_rails.html) should boost application performance. There are several types of cache storage available, each with a different level of Docker compatibility.

If the directory *tmp/cache* is present, [FileStore](http://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-filestore) is used by default. FileStore is not well-suited for use with Docker containers; it may degrade performance or even corrupt data in multi-container setups with shared data volumes.

[MemoryStore](http://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-memorystore) (which gives each process its own fast cache in ­memory) and [MemcacheStore](http://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-memcachestore) (which has an external cache that is shareable across multiple containers and processes) are better options.

Based on the presence of an environment variable with the address of an external *memcached* service, the suggested configuration provides an automatic choice between two recommended options. 

#### config/application.rb


```ruby
# please, abstain from overriding this setting in environment/*.rb 
 
if ENV["RAILS_CACHE_MEMCACHED_HOST"].present?  
    config.cache_store = :mem_cache_store, ENV["RAILS_CACHE_MEMCACHED_HOST"] 
else 
    config.cache_store = :memory_store 
end  
```


The *memcached* service can be run as a Docker container with an [official image](https://hub.docker.com/_/memcached/).

# Static assets

The alternative strategies for serving static assets depend on the environment and available infrastructure. However, building a single Docker image that is compatible with all common use cases is possible.

## Step 1. Enable global static file serving 
The appropriate option has a different name in different Rails releases. Fortunately, the gem [rails_serve_static_assets](https://github.com/heroku/rails_serve_static_assets) encapsulates the diversity, ultimately setting the required value to true regardless of the framework version.


#### Gemfile

```ruby
gem 'rails_serve_static_assets' 
```

## Step 2. Include precompiled assets in the image
To precompile assets at build time, add the following instruction to the end of the Dockerfile: ``COPY . /usr/src/app``:


#### Dockerfile

```Dockerfile
RUN rake assets:precompile
```

## Step 3. Enable assets debugging in development
To simplify debugging and modifications, the application should serve assets unconcatenated while in development mode.

Ensure the following line is included in *development.rb* **only**.


#### config/environments/development.rb

```ruby
config.assets.debug = true 
```

## Step 4. Allow a configurable CDN host for production

In some production setups, the assets URL should refer to the external CDN instead of the application domain. Let's make this setting configurable.


#### config/environments/production.rb

```ruby
if ENV["PRODUCTION_ASSET_HOST"].present? then 
    config.asset_host = ENV["PRODUCTION_ASSET_HOST"] 
end 
```

## Step 5. Ensure a unified assets prefix 

Redefining *config.assets.prefix* is not required under normal conditions, so the appropriate setting does not appear in the configuration files. If the prefix must to be customized, make sure that the new parameter is global.

# Logging

Logging is another problem that has many alternative solutions. The optimal choice may depend on many factors, such as the available log processing infrastructure and the constraints imposed by the target deployment platform.

Redirecting logging output to STDOUT is the simplest and most universal solution. Docker supports and recommends this behavior, which is compatible with the [Twelve-­factor principle of treating logs as event streams](http://12factor.net/logs).

The gem [rails_stdout_logging](https://github.com/heroku/rails_stdout_logging) enforces appropriate Rails configuration and prevents STDOUT buffering. It should not, however, be used with Rails 5.

#### Gemfile

```ruby
gem 'rails_stdout_logging' 
```

Logging should be explicitly configured for Rails 5:

#### config/application.rb

```ruby
logger  = ActiveSupport::Logger.new(STDOUT) 
logger.formatter = config.log_formatter 
config.logger = ActiveSupport::TaggedLogging.new(logger) 
```

# IP tracking

In a typical cloud setup, the various reverse proxies that are preprocessing inbound traffic can obscure the actual client IP.

If Nginx acts as reverse proxy, make sure that it is configured to set [X­Forwarded­For header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#var_proxy_add_x_forwarded_for):

#### nginx.conf

```
# this line should appear next to the proxy_pass directive 
proxy_set_header X­Forwarded­For $proxy_add_x_forwarded_for; 
```

Rails can automatically [recognize the header](http://api.rubyonrails.org/classes/ActionDispatch/RemoteIp.html).

Informing Rails about proxies used in your infrastructure is also advisable. The code below reads a JSON-formatted list of known proxy networks from an environment variable. Then, it extends the default array of trusted proxies accordingly.

#### config/application.rb

```ruby
# Be aware, for Rails versions older than 4.2 code may need some adjustments. 
if ENV["TRUSTED_PROXIES"].present?  
    require 'ipaddr' 
    config.action_dispatch.trusted_proxies = Array.wrap(  
        JSON.load(ENV["TRUSTED_PROXIES"])  
        ).map { |x|  
        IPAddr.new(x) 
        } + ActionDispatch::RemoteIp::TRUSTED_PROXIES
end  
```

# Loading environment­-specific gems

In the article [Dockerizing your Rails application](../creating-rails-dockerfile/), we recommended installing all available gems (i.e., not providing any `--without` options to the `bundle install` command) in the image. At the cost of a slight increase in size, this technique allows the same image to be used in different environments.

Rails automatically loads gems from the appropriate Bundler group (*:development*, *:test* or *:production*) along with the [default gems](http://bundler.io/v1.11/groups.html). This prevents debugging and profiling gems from being loaded in production, despite their being installed.

To prevent alteration of this behavior, the following line should be left intact:

#### config/application.rb

```ruby
Bundler.require(*Rails.groups) 
```

# Concurrent web server

By default, Rails uses the WebRick server, which does not support concurrent request processing. It is unsuitable for production, so the application image should be configured to use an alternative such as [puma](https://github.com/puma/puma) or [unicorn](https://rubygems.org/gems/unicorn).

Adding either of these to the application requires just a few steps:

* **Update the Gemfile** to include the gem ("unicorn" or "puma")
* **Create a server configuration file** (e.g., [*config/puma.rb*](https://github.com/puma/puma/blob/master/examples/config.rb)), with the following options:
    * The server should listen at the port exposed in the Dockerfile (e.g., 3000) on all interfaces.
    * Daemonize mode should be disabled.
* **Update the CMD instruction in the Dockerfile** (or the last line in [*docker/start.sh*](../multiple-processes/)), changing ``rails server`` to a server­-specific launch command:
    * If the gem is automatically recognized by running ``rails server``, this step may not be necessary for puma. 
    * If the CMD instruction is modified, keep it in [exec format](https://docs.docker.com/engine/reference/builder/#cmd) (a list of commands and arguments, not a single string).
    * If *docker/start.sh* is modified, make sure that the ``exec`` prefix is present before the server launch command.
    * Avoid daemonization options and use foreground mode.

# Summary

* Configuration differences across environments are minimized.
* The application always reads secrets from environment variables.
* For additional services (including those usually assumed to run on localhost), the default connection endpoint is a [Docker container alias](../linking-containers/). Environment variables can override these settings.
* Logging is redirected to STDOUT.
* Assets are precompiled while the image is being built, and the application is configured to serve static files.
* The default web server is replaced with the alternative, which can process requests concurrently.
