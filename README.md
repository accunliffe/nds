## node's unix domain socket server

If we observe controlled TCP/IP loopback locally and/or remotely, or simply turn our attention to the published redis project benchmarks, the fact is a unix domain socket will often reach more efficient transport levels in the order of 50% more throughput than a TCP port, [see Factors impacting Redis performance, 3rd bullet point from bottom of that section](http://redis.io/topics/benchmarks).  

Unix domain sockets use the file system as their address name space. They are [referenced by processes as inodes in the file system](http://www.linfo.org/inode.html). This allows two processes to open the same socket in order to communicate. However, communication occurs entirely within the operating system kernel.

In addition to sending data, processes may send file descriptors across a Unix domain socket connection.  

Of course, [node userland](//github.com/joyent/node/wiki/node-core-vs-userland#this-is-a-good-thing) [generally passes a port number](//github.com/visionmedia/express/blob/master/examples/cors/index.js#L42-L43) among other initializing arguments to Node's HTTP listener, and we all know this runs very fast TCP port numbered servers. However, [Ryan Dahl](http://shitryandahlsays.tumblr.com/post/33834861831/but-who-decides), [Isaacs](http://blog.izs.me/) and core provided an equally trivial mechanism for initializing the runtime via unix socket. Just pass a string directory path in leu of a port number to [Node's HTTP function](http://nodejs.org/docs/latest/api/http.html#http_server_listen_path_callback). 

A domain communication socket opens and remains at the end of our path.

The benefit of latency reduction across the stack surrounding central application logic incurs a cost of additional configuration in meeting the requirements of meaningful node process initialization. For example, we know a TCP port is available to the browser or request/response interface immediately on initializing. 

Unix Domain Sockets do not lend themselves to our common TCP port number req/res object exchange without an upstream mechansim available to pass IO routines from kernel to transport layer, such as Nginx's upstream proxy. 

To open unix style sockets, user permission at the file context is sufficient, unless the upstream proxy passes to port 80. 

Port 80 represents an aditional step to account for in meeting its root permission burden during configuration. While this is not unique to TCP port number configuration at the proxy level, once the socket is closed after having established connection to port 80, root permission becomes necessary to [remove the closed socket remaining on the file system](http://man7.org/linux/man-pages/man7/unix.7.html#NOTES), though BSD systems may ignore permissions for UNIX domain sockets.

Every time a user's server restarts, it must unlink or delete a potentially root permission locked socket descriptor left behind after socket communcation. 

Also the node process must be intialized with root permission in order to write during communication with any proxy passing to port 80, or else it will fail to write a response once an incoming request over port 80 has been written to the socket by the root user.