+++
date = '2025-03-20T10:50:03-06:00'
draft = true
title = 'Redis in Ruby'
+++

Building on some of my research into concurrency and systems programming, I have been working through a project called 'Build Your Own Redis' in [CodeCrafters](https://codecrafters.io/)  and wanted to share my experience so far and some notes about Redis and Ruby.
### What is Redis?
Redis is an in-memory storage application that is used as a key-value database, meaning that clients can associate values like strings, arrays, and sets with keys. 
### What does Redis do? 
Redis:
- runs on the internet
- accepts TCP connections
- reads commands from clients, parsing the Redis protocol
- executes those commands, potentially persisting values
- returns results of commands

The project I am going to present performs these basic actions but only responds to a few different [Redis commands](https://redis.io/docs/latest/commands/). I have yet to get to some other features of Redis like replication. Below I will get into how my project achieves each of these basic functions.
### Network
This part is pretty simple, just using the Ruby sockets library to start a TCPServer on a configurable port:
```ruby
require 'sockets'
# ...
module Redis
  class Server
    def initialize(port)
      @port = port
      @server = TCPServer.new port
      # ...
    end
  end
end
```
### Concurrency model
I presented a few ways concurrency can be done in Ruby in my last blog post, and I chose to implement an event loop for this project for a few reasons:
- I was interested in it
- Redis' actual implementation uses an event loop

Specifically, I chose to use [nio4r](https://github.com/socketry/nio4r) and the [Reactor design pattern](https://en.wikipedia.org/wiki/Reactor_pattern) to do so
- nio4r: this is an IO library for Ruby that provides a selector API, allowing applications to monitor I/O objects for readiness
- Reactor: this is a design pattern that involves defining callbacks in relation to an IO object being ready to read from or write to

This is what the concurrency implementation looks like:

```ruby
require 'nio4r'

module Redis
  class Server
    def initialize(port)
      # ...
      @selector = NIO::Selector.new
    end

    def start
      monitor = @selector.register(server, :r)
      monitor.value = proc { accept_new_client }
      loop do
        @selector.select do |monitor|
          monitor.value.call
        end
      end
    end

    def accept_new_client
      # if this is running a new client has connected 
      client = @server.accept
      # tell nio4r to monitor the client connection for reading
      monitor = @selector.register(client, :r)
      # then run this callback
      monitor.value = proc { read_command(monitor) }
    end

    def read_command(monitor)
      # read something from the client
      # then switch the interest to writing
      monitor.interests = :w
      monitor.value = proc { respond(monitor) }
    end

    def respond(monitor)
      # write something back to the client 
      # and switch back to reading
      monitor.interests = :w
      monitor.value = proc { read_command(monitor) }
    end
  end
end
```

The flow of a request-response cycle looks something like this:
1. Registering the actual TCPServer IO object with the selector means we will get notified when that is ready to read from, which means a client has arrived
	1. The callback registered for that event is `accept_new_client`, which accepts a new client, producing another IO object, and that new client is stored in the callback function `read_command`
	2. When the client connection is ready to read from, meaning a command is being sent, triggers the callback associated with that connection's IO object. When it is fully read, the interest is changed from reading to writing and the callback value is changed to `respond`.
	3. The connection being ready to read from triggers it to be selected in the main loop, so the `respond` method is called, and the interest is switched back to reading. 

You will see the IO methods in more detail coming up
### IO
Since I chose to use an event loop, I have to use non-blocking IO. A quick aside on that:
#### Blocking I/O
Think of the first couple ruby scripts you ever wrote. One of them probably involved getting **input** from the 'user' using `gets`, and printing some result, like so:
```ruby
name = gets.strip
puts "Hello, #{name}"
```
`puts` and `gets` are both blocking, meaning that the execution of your script will be suspended by the OS until data from the user is ready to be read into `name`, and/or written to the terminal.
#### Non-blocking I/O
Non-blocking I/O involves handled an error when the IO object is not yet ready. The non-blocking version of that script looks like:
```ruby
begin
  name = @stdin.read_nonblock(100)
  @stdout.write_nonblock(name)
rescue IO::WaitReadable
  retry
rescue IO::WaitWritable
end
```
The `read_nonblock` and `write_nonblock` methods available on IO objects like `$stdout` will raise an error that communicates back to the application that that IO object is not ready.
#### The Redis protocol
Reading a command for the Redis server to execute requires understanding how Redis clients will send that command. Using a pre-existing Redis client to interact with my server implementation looks like this: 
```shell-session
$ redis-cli SET foo bar
  OK
$ redis-cli GET foo
  "bar"
```
The raw command my server will receive first looks like `*3\r\n$3SET\r\n$3foo\r\n$3bar\r\n`, which the Redis spec calls an array of bulk strings. Here are the different parts of that:
- `*3`: This indicates the array will consist of 3 bulk strings
- `\r\n`: This is a line feed character, used to separate the different parts
- `$3`: This is the first part of a bulk string, and 3 refers to the bytes the following string value will consist of
- `SET`: This is a Redis command, and is three bytes long since 1 ASCII character is 1 byte
- `foo`: This is the key argument to `SET`, also serialized as a 3 byte bulk string
- `bar`: This is the value argument to `SET`, which also happens to be a 3 byte bulk string
The Redis protocol indicates the response to `SET` should be the string value `OK` serialized as a Redis Simple String, which looks like: `+OK\r\n`
The response to the `GET` command is serialized as a bulk string, which will look like this to the Redis client: `$3\r\nbar\r\n`

My implementation of a parser for the Redis protocol consists of a few parts, that are invoked from the `read_command` method in the `Server` class:
- a class to encapsulate calling non-blocking IO methods on the client connection
- a class to keep track of the progress reading a complete command
- a class corresponding to the command itself
- a class to orchestrate the classes above and build a complete Command
```ruby
require 'socket'
module Redis
  class Reader
    attr_reader :io
    def initialize(io)
      @io = io
    end

    def read_bulk_string
      len = read_token.delete("$").to_i
      read_n_bytes(len + 2)
    end

    def read_token
      buf = ""
      io.read_nonblock(4, buf)
      if buf.length > 4
        raise 'short read'
      end
      buf.delete("\r\n")
    end

    def read_n_bytes(n_bytes)
      buf = ""
      io.read_nonblock(n_bytes, buf)
      buf.delete("\r\n")
    end
  end

  class CommandState
    attr_reader :current
    # truncated for brevity, but implements a state machine 
    # using a hash
  end

  class Command
    IMPLEMENTED_TYPES = ['PING', 'ECHO', 'SET', 'GET'].freeze
    IMPLEMENTED_OPTIONS = ['px']
    attr_accessor :length,
                  :type,
                  :key,
                  :value,
                  :options,
                  :pending_option_key
  end

  class CommandBuilder
    attr_reader :reader, :command, :state
    def initialize(reader)
      @reader = reader
      @command = Command.new
      @state = CommandState.new
    end

    def build
      until state.complete?
        case state.current
        when :read_length
          command_length = reader.read_token.delete("*").to_i
          if command_length < 1
            raise BadReadError
          end
          command.length = command_length
        when :read_type
          command.type = reader.read_bulk_string
        when :read_key
          command.key = reader.read_bulk_string
        when :read_value
          command.value = reader.read_bulk_string
        when :read_option_key
          command.pending_option_key = reader.read_bulk_string
        when :read_option_value
          opt_value = reader.read_bulk_string
          command.set_option(command.pending_option_key, opt_value)
        end
        state.transition! command
      end

      command.persist!
      command
    end
  end
end
```

### Executing a Command
I'm not going to go too in depth in this section, since the most complicated thing my implementation does right now is saving a value to a hash for the SET command.

### Returning a Result to the client
Finally, my Redis implementation can return a result matching the Redis spec to the client based on which command it executed. 
```ruby
module Redis
  class Command
    # ...
    def as_bulk_string(message)
      "$#{message.length}\r\n#{message}\r\n"
    end

    def as_simple_string(message)
      "+#{message}\r\n"
    end

    def null_string
      "$-1\r\n"
    end

    def encoded_response
      case type
      when 'PING'
        as_simple_string('PONG')
      when 'SET'
        as_simple_string('OK')
      when 'ECHO'
        as_bulk_string(value)
      when 'GET'
        # if able to retrieve a value associated with the key from storage,
        # return it as a bulk string
        # else return a null string
        retrieved_value = get(key)
        retrieved_value ? as_bulk_string(retrieved_value) : null_string
      end
    end
  end

  class Server
    # ...
    def respond(monitor, command)
      monitor.io.write_nonblock(command.encoded_response)
      monitor.interests = :r
      monitor.value = proc { read_command(monitor, CommandBuilder.new(Reader.new(monitor.io))) }
    rescue IO::WaitWritable
    end
  end
end
```

### Next Steps
My full implementation is located [here](https://github.com/jakebills1/codecrafters-redis-ruby) and I am looking forward to expanding it to capture more of the full functionality that Redis has.