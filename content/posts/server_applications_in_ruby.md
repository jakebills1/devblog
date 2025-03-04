+++
date = '2025-02-28'
draft = false
title = 'Server Applications in Ruby'
+++

As a full-stack developer using Ruby on Rails, I rely on the abstraction that when an HTTP request is routed to a controller, my 'job' of writing the business logic handling that request begins. But how is it that a Rails applications is able to 'listen for' and 'accept' requests? What is necessary to build up the abstraction that we rely on as application developers? 

The answer is a server. In the term client-server programming the server is what handles requests from 1 to many clients. The server most Ruby on Rails applications use is called [Puma](https://github.com/puma/puma), and it functions by 
- listening on a particular port
- accepting TCP connections from clients that connect to that port 
- reading HTTP requests from connected clients
- invoking the Rails application to perform the business logic required to handle the request
- writing a response back to the client

Most of the things above are part of the *sockets API*, which is a set of functionality provided by the kernel to allow communication between processes. I will be focused solely on how processes on 2 different hosts can communicate over the internet.

Another thing to know about the steps above is that they are *blocking* by default. That means that when an application that is listening on a particular port calls the `accept` syscall to accept connections on that port, that process is suspended by the OS until a connection actually arrives, so the CPU can do other work. 

That also means that it is necessary for real-world server applications to be concurrent. Otherwise an incoming connection could not be accepted if the server process was busy handling another request. What follows is a presentation of a simple server application that echos a message back to the client using some of the different concurrency methods available in Ruby. 

## Non-concurrent Server

This is just to show the basic example that we are going to iterate on:
```ruby
require 'socket'  
  
class EchoServer  
  def self.run(port:)  
    server = TCPServer.new port  

    # infinite loop once listening on port
    loop do  
      # blocking call to accept to get new connection  
      @client = server.accept  
      # client has connected
  
      # blocking read from client  
      while echo = @client.gets  
        # blocking write to client  
        @client.puts echo
      end  
      @client.close
      # client disconnected  
    end  
    server.close  
  end  
end
```

This uses the [TCPServer class](https://ruby-doc.org/core/exts/socket/TCPServer.html) from the builtin gem `sockets` to encapsulate the work of setting up a TCP Server. It could be tested by:
- pasting this class into irb
- running `EchoServer.run port: <port>` where port is an unused port on your machine
- opening a TCP connection on that port in another terminal pane or tab, eg `telnet localhost <your port>`

## Process-based Server

A simple Process based server blocks with `accept` until a connection arrives, forks a process to handle that connection, and continues the loop at `accept` again. 
```ruby
class EchoServer  
  def self.run(port:)
    server = TCPServer.new port 
  
    loop do  
      @client = server.accept
  
      pid = fork do
        server.close  
        while echo = @client.gets  
          @client.puts echo   
        end  
        @client.close  
      end  
      # make sure to detach after forking process to handle client  
      Process.detach(pid)  
    end  
    server.close  
  end  
end
```

The process is one of the core abstractions to programming in general. Any program you write becomes a process while running. The `fork` method in the above example creates a child process, using the `fork` syscall under the hood. 
> By default, programs run at a lower level of privilege and are not able to directly make network connections, open files, writes files, etc. Syscalls are ways for *user-mode* programs to ask the kernel to do things requiring a higher-level of privilege. Try using `man 2 fork` to read the man page for the `fork` syscall in your terminal. 2 refers to the section of the man pages pertaining to syscalls.

I added some logging to make those processes more apparent in the output of `ps`, or `htop`:

{{< figure src="/images/echoserver_processes.png" width="600">}}

Notice that the code within the block passed to [fork](https://ruby-doc.org/core/Kernel.html#method-i-fork), which is what the child process will run, uses the same blocking IO methods as the first example. This is fine in this case, as the child process will be suspended while waiting to receive a message from the client, not the parent process. 

Concurrent programs based on processes tend to have issues scaling due to the overhead of forking a process for every connection. Real life programs do use process-based concurrency, but often *pre-forking* is used to create a pool of processes at startup.

## Threads 
Threads are another method of concurrency provided at the OS level via a syscall. They are similar to processes, however when a new thread is created it shares the memory with it's parent, where as memory is copied over to child processes. Threads are considered *lighter weight* than processes. 

```ruby
class EchoServer  
  def self.run(port:)
    server = TCPServer.new port  
  
    loop do
      client = server.accept  
        
      Thread.new(client) do
        while echo = client.gets
          client.puts echo
        end  
        client.close  
      end  
    end    
    server.close  
  end  
end
```

This implementation has a similar problem to the example using processes: the overhead of creating new threads is linear with the number of connections received. Real-life programs handle this by using a thread pool, but this raises the possibility of leaving clients waiting when all threads are busy handling slower clients.

## IO Multiplexing / Event Loops 
Many successful server designs have avoided the problems associated with using processes and threads by using non-blocking IO. This approach uses one of a variety of methods that monitor the clients a program is servicing: `select, poll, epoll, kqueue, etc`.  IO multiplexing is a fancy term for this idea, where a single-threaded program is used to handle many connections. 
My example will use the implementation included in Ruby: [IO.select](https://ruby-doc.org/core/IO.html#method-c-select). This method uses the `select` syscall under the hood, which: 
- accepts arrays of the IO objects your program reads to and writes from
- returns arrays of those IO objects that are ready to be read from or written to. 

```ruby
class EchoServer
  def initialize(port:)  
    @to_read = []  
    @to_write = []  
    @messages = {}  
    @server = TCPServer.new port  
  end
	
  def run  
    to_read << server  
    loop do
      ready_to_read, ready_to_write = IO.select(to_read, to_write)
	  
      ready_to_read.each do |socket|
        if socket == server  
          accept_new_client(socket)  
        else  
          read_from_client(socket)  
        end  
      end
	    
      ready_to_write.each do |socket|  
        write_to_client(socket)  
      end  
    end  
  ensure
    server.close
  end
	
  private
  attr_reader :to_read, :to_write, :messages, :server
	
  def accept_new_client(socket)  
    begin  
      client = server.accept_nonblock  
    rescue IO::WaitReadable  
      if !to_read.include?(socket)  
        to_read << socket
      end 
      return
    end  
    messages[client] = []  
    to_read << client  
  end
	
  def read_from_client(socket)  
    begin
      # 10000 is an arbitrary choice here,
      # and not appropriate for production use
      message = socket.read_nonblock(10000)  
      messages[socket].push(message)  
      if !to_write.include?(socket)  
        to_write << socket  
      end  
    rescue IO::WaitReadable  
      if !to_read.include?(socket)  
        to_read << socket  
      end  
    rescue EOFError  
      # this means a client has disconnected
      messages.delete(socket)  
      to_read.delete(socket)  
      to_write.delete(socket)  
      socket.close  
    end  
  end
	
  def write_to_client(socket)  
    message_queue = messages[socket]  
    until message_queue.empty?  
      message = message_queue.shift  
      begin  
        socket.write_nonblock(message)  
      rescue IO::WaitWriteable
        message_queue.unshift(message)  
        if !to_write.include?(socket)  
          to_write << socket  
        end
      end  
    end  
    to_write.delete(socket)  
  end
end
```

This implementation is definitely more complicated, so let me point out some notable differences.
- `read_nonblock` and `write_nonblock` are now used to read messages from the socket and write them back. 
  - They will throw an error is the socket is not actually ready, so errors must be rescued.
- since the main loop handles messages from many clients, some state is necessary to keep track of which clients sent which messages

Real-life server implementations would not use `IO.select` as I did. The syscall it invokes, `select` has some limitations on the total number of objects it can monitor and some performance issues. The web server Puma features an event loop using [nio4r](https://github.com/socketry/nio4r). nio4r implements the [Reactor design pattern](https://en.wikipedia.org/wiki/Reactor_pattern) as it's API and uses more modern ways to monitor I/O events than `select`. 

## Fibers
Ruby has a new way of providing developers the ability to write concurrent programs: Fibers. 

These are similar in usage to threads and processes but are managed by the programmer in user-space, not the OS. At the time of writing it is expected that a Ruby programmer wanting to use Fibers writes their own logic to coordinate their execution called a *Scheduler*, so I will be using a gem called [async](https://github.com/socketry/async) in my example that provides a fiber scheduler and some abstractions for asynchronous programming.

```ruby
require 'async'
require 'io/endpoint/host_endpoint'  
  
class EchoServer
  def initialize(port:)  
    @server = IO::Endpoint.tcp("localhost", port)  
  end  
  
  def run 
    Async do
      @server.accept do |client| 
        while message = read_from_client(client)  
          write_to_client(client, message)  
        end  
        client.close  
      end  
    end  
  end
    
  def read_from_client(client)  
    client.gets  
  end  
  
  def write_to_client(client, message)  
    client.puts message  
  end  
end
```

Observe the following in the `#run` method of this implementation:
- `Async do ... end` starts the event loop that will be managed by the `async` gem behind the scenes
- the block passed to `@server.accept` will be run asynchronously when a client connects
	- this code uses blocking IO methods (`gets` and `puts`) so the scheduler will suspend fibers waiting on IO to perform other work on one thread

## Real life applications

In practice, applications use all of the above methods, often in combination. When Puma runs in clustered mode, for example, it forks a number of child processes called workers, each worker maintains a thread pool to process requests, and a separate thread uses an event loop to read in requests off of the socket and place them in a queue for the workers to consume. 

## Conclusion

I hope this has helped you understand more of what is under the hood of programs you use every day as a Rails developer, like Puma. I am including more fleshed out versions of the examples in this article [here](https://github.com/jakebills1/ruby_concurrency_examples) for your reference.

## Sources and More Information
- [Puma Architecture Reference](https://github.com/puma/puma/blob/ca201ef69757f8830b636251b0af7a51270eb68a/docs/architecture.md)
- [Puma's use of nio4r](https://github.com/puma/puma/blob/ca201ef69757f8830b636251b0af7a51270eb68a/lib/puma/reactor.rb)
- [Working With TCP Sockets in Ruby: a much deeper look at the Sockets API available in Ruby](https://workingwithruby.com/wwtcps/intro/)
- [Async IO documentation](https://github.com/socketry/async-io?tab=readme-ov-file#usage)
- [man 2 fork](http://man.he.net/?topic=fork&section=2)
- [man 2 socket](http://man.he.net/?topic=socket&section=2)
- [man 2 select](http://man.he.net/?topic=select&section=2)
- [Unix Network Programming: Stevens](https://unpbook.com/)
