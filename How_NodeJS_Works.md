# NodeJS

- Network connections in Unix are called sockets/file descriptors - they point to objects in the kernel (read/write/close/poll etc)
- Creating a network connection involves 3 steps
  - Call socket()
  - Bind it to a port
  - Call listen()
- The accept() system call accepts incoming connections, it happens to be a blocking system call. If you request something from the kernel which it can't immediately do, your process will be descheduled and CPU resources will be taken from it. It will get scheduled again when what you requestd form the kernel is available.
- Thread per connection model for servers - traditional non event loop servers like Java servers use this approach
  - Call accept() when a new client request comes in to the server
  - accept() is a blocking system call so the process is descheduled
  - TCP connection is available, the process as unblocked and rescheduled
  - You now have a new socket for the TCP client
  - Now you can read or write to the TCP socket but while you're doing this, you can't accept new connections
  - So you start creating multiple threads, one per client connection
    - each child thread creates a new connection and does whatever compute, read/write that needs to be done on that TCP socket
    - the main thread is responsible for accepting new connections and spawning child threads
  - While you can now spawn even a thousand threads, spawning a thread is still too much work
  - All you need to know to talk to a client is
    - its socket descriptor
    - what to do with that socket descriptor
  - This is solved using epoll(linux), kqueue(BSD), Windows has other APIs
    - We have one server socket
    - We are interesting in knowing if there are incoming TCP sockets, so 
    - We create an epoll descriptor to ask the kernel if there are any incoming TCP connections
    - We do this in an epoll loop, while loop calling epoll_wait in every iteration, this blocks the kernel till a new TCP connection comes in
    - We accept the connection and get a socket descriptor for the TCP client
    - Then we add that client socket descriptor to the epoll_loop to check if any data come into that client socket
    - So now the epoll loop blocks and waits for 2 occurances:
      - new TCP connections - accpet them
      - incoming data on a specific TCP connection - read data from the socket
    - We also write client socket descriptors to teh epoll loop to check when the socket becomes writeable to write to it
    - LibUV does all this - a semi infinite while loop calling epoll_wait(), calling the kernel, blocking till something interesting happens - events, callbacks
    - If there are not events, the event loop exits as there is nothing to wait on in the epoll_wait loop. 


# What is epollable and what is not? 3 classes of things:
- Pollable file descriptors - directly epollable
  - sockets (net/dgram/http/tls/child_process/pipes/stdin,out,err)
  - signals
  - child processes
  - C++ addons can be pollable if they integrate with the loop, but they should use the thread pool if they make blocking system calls and are blocking the uv loop
- Time - one timeout is directly pollable and others are made epollable (somehow)
  - all timeouts are sorted and epoll waits on the next timeout that is about to expire
- Everything else happens off the loop and signals back to the loop when done
  - file system - even though it is a file descriptor it cannot be epolled, so this happens in the uv thread pool
  - blocking call is made by a thread in the uv thread pool and when it completes, readiness signal is sent back to the event loop using an eventfd or a self-pipe (you can't even wait on threads in epoll)
  - dns.lookup() happens in the uv thread pool
  - all other dns operations are integrated in the uv loop


# So the uv thread pool is used by
- fs
- dns.lookup()
- crypto.randomBytes(), crypto.pbkdf2()
- http.get and http.request() only uses it if called with a hostname which needs resolution using dns.lookup() which uses it
- any C++ addons if they use the thread pool, but if they make blocking sytem calls here, you can have thread pool contention
4 threads by default, UV_THREADPOOL_SIZE = 4

## Track your event loop time, if it is high, you are either doing something CPU intensive or you are making blocking system calls in the uv loop
