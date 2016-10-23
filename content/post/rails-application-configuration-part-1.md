+++
date = "2016-10-20T02:16:04Z"
draft = false
title = "Rails Application Configuration Part 1"

+++

A number of years ago, working in ASP.NET, when I wanted to make add configuration to web applications,
I knew of two main options: Web.config, and the database. The former (an XML file) was pretty much only
good for database connection strings. Using the database was hardly surprising then.  The database was
easy for applications to read from, and could be written to (i.e. configuration changed on a running
application) with any SQL client.

In Rails we have different options. Rails itself provides several places: _application.rb_ and the
environment-specific files like _development.rb_ and _production.rb_, as well as initializers under
_config/initializers_. These are obviously just ruby and lead to storing configuration in singletons,
either `Rails.configuration` or a global like `$redis`. While where exactly to load different configuration
is an interesting topic, in this post I'm more concerned with where the values come from. The simplest
and most common are:

1. Directly in ruby, in _production.rb_, for example, like:

```ruby
config.deliver_email_digest = :weekly
```

2. In a YAML file, _config/redis.yml_ for example:

```
development:
  host: localhost
  port: 6379
  database: 0
production:
  host: 10.0.0.17
  port: 6379
  database: 1
```

3. In an environment variable inside ruby, _config/initializers/redis.rb_ for example:

```
$redis = Redis.new(host: ENV["REDIS_HOST"], port: ENV["REDIS_PORT"], db: ENV["REDIS_DATABSE"])
```

4. Using ERB inside a YAML file to load an environment variable, like Rails does for _secrets.yml_:

```
development:
  secret_key_base: abcd
test:
  secret_key_base: efgh
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

Directly putting values in ruby like `:weekly` from the first example has fallen out of favour, in no small
part due to the influence of the [Twelve-Factor App](https://12factor.net/config), which also
highlights the issues of managing standalone files (like the above-mentioned _config/redis.yml_). Not wanting
to commit sensitive database credentials to version control, and the need for more flexible yet simple
configuration combinations do indeed call for different strategies.  The author of
that guide, Adam Wiggins, was at Heroku, so it's natural that he recommends environment variables. Heroku
runs applications in containers which lends itself nicely to this practice. A container runs a single process
and setting the environment for this process is straightforward. Heroku provides both web and cli tools
for setting environment variables, so it's a great option if you're running your application there. Docker,
similarly, allows you to set environment variables in your _Dockerfile_:

```
ENV POSTGRES_URL="postgres://username:password@10.0.1.15/database"
```

If you're not running your application in containers, you can still use environment variables, but it's
not as straightforward, and there are some issues to be aware of. In development, you _can_ run your
server like this:

```
REDIS_URL="redis://localhost:6379/3" rails server
```

As you can imagine, no doubt, this quickly becomes unmanageable. That's where the [dotenv](https://github.com/bkeepers/dotenv) gem
comes in. It allows you to load environment variables from a simple `.env` file. The [foreman](https://github.com/ddollar/foreman) gem similarly has the capability to load environment variables from the `.env` file (or another
file specified in a command-line switch). Using foreman also has the benefit of being able to export your processes into Ubuntu upstart scripts. These scripts will also include environment variables from `.env` which is very convenient
since upstart-run processes start with a mostly clean environment.

Another particular challenge is zero-downtime deploys with a forking web server like unicorn. Replacing the running unicorn with a reload doesn't give you an opportunity to reload environment variables. Workarounds like [this](https://github.com/intercity/chef-repo/issues/159#issuecomment-224311594) do work, but at this point you're loading environment variables (in production) from a file, so it's
essentially no different than loading your configuration directly from a file
instead of from the environment at this point.

If you are using disposable containers on Heroku or with Docker, where updates are done through creating and destroying containers rather than have an existing master process exec a new master process, environment variables make perfect sense. Otherwise
you might as well stick to reading local YAML files since you'll probably need
to explicitly reload environment variables from a file anyway.

In part 2, I'll examine application configuration with distributed key-value
stores. In other words, configuration in a database is back, provided it's
not a relational database.
