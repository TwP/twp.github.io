---
title: 'Preforking Workers in Ruby'
slug: 'preforking-workers'
date: '2011-01-02'
aliases:
  - /2011/01/preforking-workers
draft: false
---
On the same day, two separate rubyists asked me the very same question: "How
do you communicate between the parent and a forked child worker". This
question needs a little background information to be properly understood.

Pre-forking is a UNIX idiom. When a process is expected to handle many tasks
simultaneously, child processes can be created to offload the work from the
parent process. Generally this makes the application more responsive; the
child processes can use multiple CPUs and handle IO streams without blocking
the parent. Eric Wong's [Unicorn](http://unicorn.bogomips.org/) web server
uses child processes in this fashion. Ryan Tomayko has a fantastic [blog
post](http://tomayko.com/writings/unicorn-is-unix) describing Unicorn and
pre-forking child processes.

[Servolux](https://github.com/TwP/servolux) provides a [Prefork
class](http://rdoc.info/gems/servolux/0.9.5/Servolux/Prefork) to manage a
pool of forked child processes. Internally the **Prefork** class uses a
pipe to send messages and status between the parent and child. This pipe
provides all kinds of niceties like heartbeat, child timeout, and rolling
restarts. However, this pipe *cannot* be used for general communication to the
child by the end user.

In fact, communication with a specific child process is usually not the
desired behavior.

Two of the major reasons for using child processes is (1) to take advantage of
multiple CPUs or (2) to handle IO intensive tasks. In both of these situations
each child process should be interchangeable with any other child process.
That is, we don't care which child is handling a calculation or some IO
process; one child is as good as another.

So, the communication pipe used by the **Prefork** class is not really what
we're after. It is used to manage a specific child. Instead we need a way to
send a message to the entire pool of child workers. Any currently available
child can handle the message.

## Beanstalkd

The simplest method of communication with the child processes is via a message
queue. The Servolux gem provides some [example
documentation](https://github.com/TwP/servolux/blob/master/examples/beanstalk.rb)
showing how to use a [Beanstalkd](http://kr.github.com/beanstalkd/) message
queue to send jobs to the child processes. Each child establishes a connection
to the queue and waits for messages to process. The user pushes messages onto
the queue to be handled by some child.

## Sockets

A harder (but more educational) method is to use sockets for communication to
a child process. The following example is lifted mostly from Ryan Tomayko's
blog post mentioned above, but with a smattering of [Servolux](https://github.com/TwP/servolux)
thrown in for good measure.

The key thing to take away here is that we create a UNIX server socket in the
parent process, and the forked children "accept" on this socket to receive
messages. As odd as it seems, the parent then creates a UNIX socket in order
to send messages to the children; the parent sends messages to the children
who are accepting.

```ruby
require 'socket'
require 'open-uri'

require 'rubygems'
require 'servolux'
require 'nokogiri'

PATH = '/tmp/web_fetch.socket'

# Create a UNIX socket at the tmp PATH
# Runs once in the parent; all forked children inherit the socket's
# file descriptor.
$acceptor = UNIXServer.new(PATH)

# This module defines the process our forked workers will run. It listens on
# the socket and expects a single URL. It will then fetch this URL and parse
# the contents using nokogiri.
module WebFetch
  def execute
    if IO.select([$acceptor], nil, nil, 2)
      socket, addr = $acceptor.accept_nonblock
      url = socket.gets
      socket.close

      doc = Nokogiri::HTML(open(url)) { |config| config.noblanks.noent }
      $stderr.puts "child #$$ processed #{url}"
      $stderr.flush
    end
  rescue Errno::EAGAIN, Errno::ECONNABORTED, Errno::EPROTO, Errno::EINTR
  end

  def after_executing
    $acceptor.close
  end
end

# Spin up a pool of these workers
pool = Servolux::Prefork.new(:module => WebFetch)
pool.start 3

# 'urls.txt' is a simple text file with one URL per line
urls = File.readlines('urls.txt')

begin
  # Keeping sending URLs to the workers until we have run out of URLs
  until urls.empty?
    client = UNIXSocket.open(PATH)
    client.puts urls.shift
    client.close
  end

rescue Errno::ECONNREFUSED
  retry

ensure
  # Give the workers time to complete their current task
  # and then stop the pool
  sleep 5

  pool.stop
  $acceptor.close

  File.unlink if File.socket?(PATH)
end
```

Because each child has a copy of the UNIX server socket, each child also needs
to close this socket. This is done in the **after_executing** method. This
method is called just before the child process exits. Resource cleanup happens
here.

The parent process also needs to close the UNIX server socket and remove the
socket file created in the tmp folder. These final steps are performed in the
*ensure* block to ensure they happen.

## Conclusion

The main concept to take away here is that pre-forked workers are
indistinguishable from one another. One child process is as good as another.
The vast majority of the time you will need to pass messages to the first
available child worker. If you find yourself needing to communicate with a
specific child then **Prefork** is most likely *not* the solution you are
looking for.
