Health.js
=========

Network service for getting CPU of remote system and receiving CPU
usage alerts.

Tested on Node.js 0.2.3 and 0.2.4. This will only run on Linux.
OSX and BSD are not supported.

Another fun project sponsored by [Last.VC](http://last.vc/)

* *Author: [Brendon Crawford](mailto:brendon@last.vc)*
* *See: [Calculating CPU Usage from /proc/stat](http://colby.id.au/node/39/)*
* *Homepage: [Health.js](https://github.com/last/healthjs)*

Basic Usage
-----------

To get help, you can run:

    $ node health.js --help

Health.js serves 2 primary modes: "streaming mode" and "event mode".
Streaming mode allows a  client to connect and receive streaming CPU usage
data. Event mode enables Health.js to notify a remote server when CPU usage
hits a certain threshold. Both modes can be run simultaneously.

Running in Streaming Mode
-------------------------

To run health.js in streaming mode, the command format is:

    $ node health.js --listen IP [--port PORT] [--cycle-time NUMBER]

Examples:

    $ node health.js --listen 127.0.0.1
    $ node health.js --listen 127.0.0.1 --port 37778
    $ node health.js --listen 127.0.0.1 --port 37778 --cycle-time 4000

Explanation of options:

*   *--listen IP*

    This is the IP address that the streaming server should listen on.
    If this option is not provided, the streaming server will not be
    enabled.

*   *--port PORT*

    This is the port number that the streaming server should listen on.
    The default is 37778.

*   *--cycle-time NUMBER*

    This is the amount of milliseconds that the server should wait
    before calculating a new CPU check. Lower numbers have a higher
    performance overhead. The default is 6000 (6 seconds).

Running in Event Mode
---------------------

To run health.js in event mode, the command format is:

    $ node health.js --remote-host IP [--remote-port PORT] \
    >      [--threshold-cpu NUMBER] [--cycle-time NUMBER] \
    >      [--threshold-cycles NUMBER] [--resend-wait NUMBER] 

Examples:

    $ node health.js --remote-host 192.168.1.2 --remote-port 37779 \
    >      --threshold-cpu 60 --threshold-cycles 20 \
    >      --resend-wait 30 --cycle-time 4000

Explanation of options

*   *--remote-host IP*

    This is the remote IP address to notify when CPU hits a certain threshold.

*   *--remote-port PORT*

    This is the port of the remote host to notify when CPU hits a certain
    threshold.

*   *--threshold-cpu NUMBER*

    This is the number that average CPU usage needs to hit to trigger
    an event notification to remote host. The default for this is 80 (80%).

*   *--cycle-time NUMBER*

    This is the amount of milliseconds that the server should wait
    before calculating a new CPU check. Lower numbers have a higher
    performance overhead. The default is 6000 (6 seconds).

*   *--threshold-cycles NUMBER*

    This is the number of CPU checks that will be used to calculate CPU average.
    The time interval of one single CPU check is determined by "--cycle-time".
    A lower number will yield more volatile results at a quicker pace.
    A higher number will yield more precise results at a slower pace.
    For example, if "--cycle-time" is set to "4000", and "--threshold-cycles" is
    set to 20, and "--threshold-cpu" is set to "60", this would mean that a
    notification would be sent if over a period of 80 seconds, the average CPU
    usage was 60%. The default for this is 10.

*   *--resend-wait NUMBER*

    This is the amount of time in minutes that health.js should wait before sending
    another notification to remote server. The default for this is 360 (6 hours).

Running in Event and Streaming Mode Simultaneously
--------------------------------------------------

Both modes can be used at the same time.

Examples:

    $ node health.js --listen 127.0.0.1 --port 37778 \
    >     --remote-host 192.168.1.2 --remote-port 37779 \
    >     --threshold-cpu 60 --threshold-cycles 20 \
    >     --resend-wait 30 --cycle-time 4000

Connecting with Client to Health.js Streaming Service
-----------------------------------------------------

A client can connect to the health.js Streaming service via tcp.

A client should connect via tcp to the listening port. The default
port is 37778 and the default interface to listen on is 127.0.0.1. To listen
on all interfaces specify "0.0.0.0" as the listening IP. Once connected, the
client may send one of two messages:

*   *get cpu once*

    This will grab one cpu update an exit

*   *get cpu loop*

    This will indefinitely grab cpu updates until client disconnects

Here are some examples of connecting via common UNIX utilities:

    ## Get one single cpu status using netcat
    $ echo "get cpu once" | nc -q -1 localhost 37778

    ## Get infinite loop of cpu status, using netcat
    $ echo "get cpu loop" | nc -q -1 localhost 37778

    ## Get infinite loop of cpu status, using telnet
    $ telnet localhost 37778
    $ telnet> get cpu loop

The response from the streaming server will be a CPU_USAGE string. See
below for more information on the CPU_USAGE string.

The CPU_USAGE string
--------------------

The CPU_USAGE string from health.js will be 2 or more decimal (floating point)
numbers separated by a space on a single line. An example response
might look like this:

    4.19 6.50 2.44 2.97 4.88

Column #1 is the average percentage of CPU usage for all cores/processors
in the system. All columns after #1 represent the CPU usage of that
particular processor/core. So, in the example above, the breakdown would be:

* System Average: *4.19%*
* Processor #1: *6.50%*
* Processor #2: *2.44%*
* Processor #3: *2.97%*
* Processor #4: *4.88%*

Receving notifications from the health.js Event service
-------------------------------------------------------

If CPU usage reaches a certain threshold, health.js will send an event
notification to "--remote-host" if it is set.

The event notification will have the following format:

    COMMAND|ELAPSED_TIME|CPU_AVERAGE|CPU_USAGE

Example Notification:

    put cpu|30000|14.33|17.28 3.11 4.75 55.32 6.12

Explanation of parameters:

*   *COMMAND*

    This will always be "put cpu".

*   *ELAPSED_TIME*

    This will be the amount of time that CPU usage was calculated over
    to calculate CPU_AVERAGE

*   *CPU_AVERAGE*

    This is the average CPU average load over the period of time defined by
    ELAPSED_TIME.

*   *CPU_USAGE*

    For information on the CPU_USAGE string, see the "CPU_USAGE" section above.

Here are some examples of setting up notification recievers via
common UNIX utilities:

    ## Receive events using netcat
    nc -kl 127.0.0.1 37779

    ## Receive events using python
    import SocketServer
    class handler(SocketServer.StreamRequestHandler):
        def handle(self):
            print self.rfile.readline().strip()
    server = SocketServer.TCPServer(('127.0.0.1', 37779), handler)
    server.serve_forever()

Security Concerns
-----------------

This service provides no means of authentication. It should only be
run on private interfaces/IPs which are only exposed to your local trusted
network. It should never be run on a public-facing IP or an IP on an un-trusted
network. If you need to run this on an untrusted network, or if you want an
an authentication mechanism, you should have it listen on  host 127.0.0.1 and access it
via an SSH tunnel.

If you don't know what any of this means, you probably should not be using
this utility.

