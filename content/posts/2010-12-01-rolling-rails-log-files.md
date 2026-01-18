---
title: 'Rolling Rails Log Files'
slug: 'rolling-rails-log-files'
date: '2010-12-01'
aliases:
  - /2010/12/rolling-rails-log-files
draft: false
---
There is a small issue with the default Rails logging setup. If left unchecked,
the production log file can grow to fill all available space on the disk and
cause the server to crash. The end result is a spectacular failure brought on
by a minor oversight: that Rails provides no mechanism to limit log file
sizes.

Periodically the Rails log files must be cleaned up to prevent this from
happening. One solution available to Linux users is the built-in **logrotate**
program. Kevin Skoglund has written a blog post describing how to use
logrotate for [rotating rails log files](http://www.nullislove.com/2007/09/10/rotating-rails-log-files/).
The advantage of logrotate is that nothing needs to change in the Rails
application in order to use it. The disadvantage is that the Rails app should
be halted to safely rotate the logs.

Another solution is to replace the default Rails logger with the Ruby
[logging](http://github.com/TwP/logging) framework. The logging framework is
an implementation of the Apache
[Log4j](http://logging.apache.org/log4j/1.2/) framework in Ruby. Although more
complex than using logrotate, the **logging** framework allows Rails
to roll it's own log files without needing to halt the application. Another
advantage is that logging destination (a file in this case) can be pointed to
a syslog daemon via a configuration file.

## Configuring Rails

Rails provides for application configuration via the `config/environment.rb`
file; setting the `Rails.logger` in this file is the normal way of specifying
an alternative logger to use. However, the logging framework needs to be
interposed much earlier in the Rails initialization process. This is
accomplished via the much loved/dreaded monkey-patch.

The following ruby code should be saved in your Rails app as
`lib/logging-rails.rb`:

```ruby
require 'logging'

class Rails::Initializer
  def load_environment_with_logging
    load_environment_without_logging

    _fn = File.join(configuration.root_path, 'config', 'logging.rb')
    return unless test(?f, _fn)

    _log_path = File.dirname(configuration.log_path)
    FileUtils.mkdir _log_path if !test(?e, _log_path)

    config = configuration
    eval(IO.read(_fn), binding, _fn)

    silence_warnings {
      Object.const_set("RAILS_DEFAULT_LOGGER", Logging::Logger[Rails])
    }
  end

  alias :load_environment_without_logging :load_environment
  alias :load_environment :load_environment_with_logging

  # Sets the logger for Active Record, Action Controller, and Action Mailer
  # (but only for those frameworks that are to be loaded). If the framework's
  # logger is already set, it is not changed.
  #
  def initialize_framework_logging
    frameworks = configuration.frameworks &
                 [:active_record, :action_controller, :action_mailer]

    frameworks.each do |framework|
      base = framework.to_s.camelize.constantize.const_get("Base")
      base.logger ||= Logging::Logger[base]
    end

    ActiveSupport::Dependencies.logger ||= Logging::Logger[ActiveSupport::Dependencies]
    Rails.cache.logger ||= Logging::Logger[Rails.cache]
  end
end
```

Include this file at the top of your `config/environment.rb` file just after
the Rails bootstrap require line.

```ruby
# Be sure to restart your server when you modify this file

# Specifies gem version of Rails to use when vendor/rails is not present
RAILS_GEM_VERSION = '2.3.8' unless defined? RAILS_GEM_VERSION

# Bootstrap the Rails environment, frameworks, and default configuration
require File.join(File.dirname(__FILE__), 'boot')
require 'logging-rails'

Rails::Initializer.run do |config|
  # Settings in config/environments/* take precedence over those
  # specified here. Application configuration should go into files
  # in config/initializers -- all .rb files in that directory are
  # automatically loaded.

  ...
end

Logging.show_configuration if Logging.logger[Rails].debug?
```

As a bonus, at the very end of the environment file is a line that will dump
the current logging configuration to **STDERR** when the Rails logger is set
to debug mode. This visual display of the configuration is very useful for
understanding where log messages will be sent.

## Configuring Logging

Now that Rails has been cowed into submission, it is time to configure the
logging framework and enable rolling log files. A new configuration file has
been introduced by the `lib/logging-rails.rb` file. The `config/logging.rb`
file contains the base settings for the logging framework; these settings can
be overridden and refined in the environment specific Rails configuration
files.

Copy the following code to your config folder:

```ruby
# When an object passed as the logging message, use the inspect method to
# convert it to a string. Other format options are :string and :yaml.
#
Logging.format_as :inspect

# Use a layout pattern for injecting a timestamp, log level, and classname
# into the generated log message.
#
layout = Logging.layouts.pattern(:pattern => '[%d] %-5l %c : %m\n')

=begin
Logging.appenders.stdout(
  'stdout',
  :auto_flushing => true,
  :layout => layout
)
=end

# Configure the rolling file appender. Roll the log file each day keeping 7
# days of log files. Do not truncate the log file when it is first opened.
# Flush all log messages immediately to disk (no buffering).
#
Logging.appenders.rolling_file(
  'logfile',
  :filename => config.log_path,
  :keep => 7,
  :age => 'daily',
  :truncate => false,
  :auto_flushing => true,
  :layout => layout
)

# Set the root logger to use the rolling file appender, and set the log level
# according to the Rails configuration.
#
Logging.logger.root.level = config.log_level
Logging.logger.root.appenders = %w[logfile]


# Under Phusion Passenger smart spawning, we need to reopen all IO streams
# after workers have forked.
#
# The rolling file appender uses shared file locks to ensure that only one
# process will roll the log file. Each process writing to the file must have
# its own open file descriptor for flock to function properly. Reopening the
# file descriptors after forking ensures that each worker has a unique file
# descriptor.
#
if defined?(PhusionPassenger)
  PhusionPassenger.on_event(:starting_worker_process) do |forked|
    Logging.reopen if forked
  end
end
```

This file is Ruby code that calls the configuration hooks provided by the
logging framework. There are many more
[examples](https://github.com/TwP/logging/tree/master/examples/) demonstrating
the various appenders and techniques to achieve the desired logging output. A
brief overview of what is happening is warranted, though.

The first line describes how objects will be formatted when passed as the log
message. The allowed values for `format_as` are *:string*, *:inspect*, and
*:yaml*.

The second line defines how log messages will be formatted before being sent
to the appenders (appenders do the actual writing of the log message to the
logging destination). The [PatternLayout](https://github.com/TwP/logging/blob/master/lib/logging/layouts/pattern.rb)
class is well documented. This layout will be assigned to the rolling file
appender.

Next comes the rolling file appender definition. Each appender is given a name
that can be used to refer to the appender later in the configuration. The
rolling file appender is configured to roll daily and keep the last 7 days of
log files. Setting the `:truncate` flag to `true` will cause the current log
file to be truncated when it is opened; usually this is not the desired
behavior when Rails start. The `:auto_flushing` flag can either be `true` or
it can be a number; when a number is used, then that number of log messages
are buffered and then written to disk in bulk. The configuration shown flushes
each log message to disk as soon as it is sent to the appender.

Finally is the configuration of the root logger. Since the **logging**
framework is written in the spirit of Log4j, there is a hierarchy of logger
instances each with their own log level and possibly their own appender. The
rolling file appender is assigned to the root logger, and the log level is set
according to the Rails configuration.

Congratulations! Rails is now rolling it's log files.

## Bonus!

Now that the **logging** framework has been integrated into the Rails app
other appenders can be used. Multiple appenders can be used simultaneously,
too.

At work, we are using the syslog appender to send all our log messages to a
central logging server. We then use [Splunk](http://www.splunk.com/) to
extract information from the aggregated log files. That is another entire post
in itself. Please respond on [Twitter](http://twitter.com/pea53) if you are
interested in learning more.

## Postscript

This approach to rolling Rails log files only works with Rails 2 projects.
Rails 3 has drastically changed the initialization process. A post is
forthcoming on how to integrate the **logging** framework with Rails 3; the
first step is digging into the new initialization process.
