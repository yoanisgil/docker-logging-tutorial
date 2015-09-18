##Introduction

Logging is a central piece to almost every application. It is one of those things you want to keep around specially when things go wrong. Much and more have been say and done on this particular subject, however, today I want to focus on [Docker](http://www.docker.com) and the options available when it comes to logging in the context of containerized application. Logging capabilities available in the Docker project are exposed in the form of drivers, which is very handy since one gets to choose how and where log messages should be shipped. As of the moment of writing this article there are five of them:

- `json-file`: This is the default driver. Everything gets logged to a JSON-structured file
- `syslog`: Ship logging information to a syslog server
- `journald`: Write log messages to journald (journald is a logging service which comes with  [systemd](http://www.freedesktop.org/wiki/Software/systemd/))
- `gelf`: Writes log messages to a [GELF](https://www.graylog.org/resources/gelf/)  endpoint like Graylog or Logstash
- `fluentd`: Write log messages to [fluentd](http://www.fluentd.org/)

This article is the first of a few ones centered around logging in Docker. We will take a rather in-depth look to the first two drivers *json-file* and *syslog*. We will see how they work and what their strengths and limitations are. 

The examples given throughout the article were created and tested using Docker v1.8 which introduces the `gelf` and `fluentd` drivers alone with a few nice additions to the `syslog` one (tagging, logging facility, etc). You can read more about it [here](https://github.com/docker/docker/blob/v1.8.1/CHANGELOG.md#180-2015-08-11).

**NOTE**: This article's source code alone with the code used can be found on [Github](https://github.com/yoanisgil/docker-logging-tutorial)

##Docker Logging 101

Before getting into the specifics of any of the available logging drivers, there are these questions you might have already asked yourself:

- What messages are actually logged? 
- How do I get my messages shipped to the logging driver?

The answer to first question it's actually dead easy: anything that gets written out to the container's standard output/error will get logged. As to the answer to the second one, you don't need to do much, at least not by default. When a new container is created, and provided that no logging specific options were passed at the moment of creating it,  the docker engine will configure the default logging driver which is the `json-file` one. So if we put together 1 and 2 it's clear that anything your application write to it's standard output/error will get written in a json file. That simple.

Ok, that last one paragraph was too long, so enough wording and let's see it in action. First we will create a simple python script which writes to both it's stdout and stderr output:

```python
 import sys  
 import time  
 
 while True:  
   sys.stderr.write('Error\n')  
   sys.stdout.write('All Good\n')  
   time.sleep(1)  
``` 
Save this to a file `logging-01.py` (or you can get it from [here](logging-01.py)) and then run:

    $ docker run --name logging-01 -ti -d -v $(pwd):/tmp  -w /tmp python:2.7 python logging-01.py
    $ docker logs -f logging-01
    
which will output:

```bash
Error
All Good
Error
All Good
```
This illustrates how messages written to `stdout/stderr` are logged and it's actually available with the `docker logs` command. But wait, isn't this the `json-file` driver? So where is this so called JSON file? Let's inspect our container and see what we can find:

    $ docker inspect logging-01 | grep LogPath
        "LogPath": "/mnt/sda1/var/lib/docker/containers/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1-json.log",
        
you will get a different output from the command above but the important thing here is that we have the path to the file where our log messages are written to. Let's take a look at this file:

    tail -2 /mnt/sda1/var/lib/docker/containers/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1-json.log
    {"log":"Error\r\n","stream":"stdout","time":"2015-09-13T03:36:30.120234597Z"}
    {"log":"All Good\r\n","stream":"stdout","time":"2015-09-13T03:36:30.120475567Z"}
    
which is self-explanatory me thinks ;). The `json-file` driver takes two more options:

- --log-opt max-size=[0-9+][k|m|g]
- --log-opt max-file=[0-9+]

about which you can read more [here](https://docs.docker.com/reference/logging/overview/). These options lets you control the rate at which log files are allowed to grow. To illustrate this lets re-run our little application:

    $ docker run --rm --log-opt max-file=2  --log-opt max-size=2k  --name logging-02 -ti  -v $(pwd):/tmp  -w /tmp python:2.7 python logging-01.py
    Error
    All Good
    Error
    All Good
    ....
here we have indicated that we only want to keep 2 log files and the max allowed size for any given log file is 2k. You can verify that this is indeed happening by grabbing the location of the default log file:

    $ docker inspect logging-02 | grep LogPath
           "LogPath":"/mnt/sda1/var/lib/docker/containers/04a96c05121777eebe6a38d63fd657f6fb6c8b9632fee7d81ccc0ff45023aedd/04a96c05121777eebe6a38d63fd657f6fb6c8b9632fee7d81ccc0ff45023aedd-json.log",
    $ cd /mnt/sda1/var/lib/docker/containers/04a96c05121777eebe6a38d63fd657f6fb6c8b9632fee7d81ccc0ff45023aedd/
    $ watch 'ls -lh *json*'

##The Syslog driver

The `json-file` driver is as simple as it gets and while it's very useful for development purposes but it won't do the job  once the list of running containers starts to growth. This is not strictly related to performance issues but rather to the fact that as the number of containers and applications starts to increase it becomes a no men work to track logging information. Enter the `syslog` driver. 

This is one of the many logging drivers which allows you to send  messages to local or remote server by means of a network protocol. So lets get ourselves a syslog server .... running in a container of course ;):

    $ docker run -d -v /tmp:/var/log/syslog -p 127.0.0.1:5514:514/udp  --name rsyslog voxxit/rsyslog
    
(**NOTE**: We're using [rsyslog](http://www.rsyslog.com/) which is an implementation of the syslog protocol)

and now lets launch our sample application but this time we will specify the logging driver to be used:

    $ docker run --log-driver=syslog --log-opt syslog-address=udp://127.0.0.1:5514 --log-opt syslog-facility=daemon --log-opt syslog-tag=app01   --name logging-02 -ti -d -v $(pwd):/tmp  -w /tmp python:2.7 python logging-01.py

Let's break down into pieces this last command line:

- `syslog-tag=app01`: A tag to apply to all messages coming from this container.
- `syslog-facility=daemon`: The syslog facility to be used
- `--log-driver=syslog`: We're explicitly configuring our container to use the `syslog` driver.
- `--log-opt syslog-address=udp://127.0.0.1:5514`: This indicates which syslog server we want to ship messages to and whether it should be done using TCP or UDP.

(*NOTE*: If you need a crash course on syslog you can check the [Wikipedia entry](https://en.wikipedia.org/wiki/Syslog) or [this link](https://blog.logentries.com/2014/08/what-is-syslog/))

It's time to check if log messages are making it to their final destination. If you're like me, you will go and type:

    $ docker logs -f logging-02
    "logs" command is supported only for "json-file" logging driver (got: syslog)

which will totally make sense once you give it a 5 seconds thought. There is no possible way, nor it should try to,  the `docker logs` could work as we expected given that messages are sent to a potentially remote server. Instead we need to look into the `rsyslog` container which is acting as our syslog server:

    $ docker exec  rsyslog tail -f /var/log/messages
    2015-09-16T01:05:17Z default docker/app01[989]: Error#015
    2015-09-16T01:05:17Z default docker/app01[989]: All Good#015

see how our `app01` tag is present on each message? Now let’s launch a new instance of our sample application but this time we will use a new tag:

    $ docker run --log-driver=syslog --log-opt syslog-address=udp://127.0.0.1:5514 --log-opt syslog-facility=daemon --log-opt syslog-tag=app02   --name logging-03 -ti -d -v $(pwd):/tmp  -w /tmp python:2.7 python logging-01.py
    $ docker exec  rsyslog tail -f /var/log/messages
    2015-09-16T01:11:27Z default docker/app02[989]: Error#015
    2015-09-16T01:11:27Z default docker/app02[989]: All Good#015
    2015-09-16T01:11:28Z default docker/app01[989]: Error#015
    2015-09-16T01:11:28Z default docker/app01[989]: All Good#015
and now we have messages coming from the new container tagged `app02`.

##What's next?

The `syslog` driver is an enhancement over the `json-file` one, specially for environments with multiple applications powered by containers, at the cost of introducing a dependency on a external service. It is also a step forward towards centralization of logging messages. However as the number of applications grow, and so it does the number of running containers, your will find your self in need of a tool letting you search for specific log messages based on a date/time range, a keyword, etc ((don't grep-me on this one please ;)).

In the next article of these series we will take a look at the `gelf` and `fluentd` driver to see how they might help to overcome some of the issues mentioned above.