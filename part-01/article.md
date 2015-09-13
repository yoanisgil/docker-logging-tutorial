Introduction
-------------------

Logging is a central piece to almost every application. It is one of those things you want to keep around specially when things go wrong. Much and more have been say and done when it comes to this particular subject.

However, today I want to focus on [Docker](http://www.docker.com) and what options are available when it comes to logging in the context of containerized application. Logging capabilities available in the Docker project are expressed in the form of drivers, which is very handy since you get to choose how and where your log messages should be shipped. As of the moment of writing this article there are five of them:

- `json-file`: This is the default driver. Everything gets logged to a JSON-structured file
- `syslog`: Ship logging information to a syslog server
- `journald`: Write log messages to journald (journald is a logging service which comes with [systemd](http://www.freedesktop.org/wiki/Software/systemd/))
- `gelf`: Writes log messages to a [GELF](https://www.graylog.org/resources/gelf/)  endpoint like Graylog or Logstash
- `fluentd`: Write log messages to [fluentd](http://www.fluentd.org/)

This article is the first of a few ones centered around logging in Docker. We will take a rather in-depth look to the first two drivers *json-file* and *syslog*. We will see how they work and what their strengths and limitations are, etc.

Docker Logging 101
----------------------

Before getting into the specifics of any of the available logging drivers, there are these questions, which you might have already asked yourself:

- What messages are actually logged? 
- How do I get my messages shipped to the logging driver?

If you asked yourself these questions and you at least gave it a try to the [daemonized hello world example](https://docs.docker.com/userguide/dockerizing/), well it turns out you already know the answers. The answer to first question it's actually dead easy: anything that gets written out to the container's standard output/error will get logged. As to the answer to the second one, you don't need to do much, at least not by default. When a new container is created, and provided that no logging specific options has been passed at the moment of creating it,  the docker engine will configure the default logging driver which is the json-file one. So if we put together 1 and 2 it's clear that anything your application write to it's standard output/error will get written in a json file. That simple.

Ok, that last one paragraph was too long, so enough wording and let's see it in action. Let's create a simple python script which writes to both it's stdout and stderr output:

```python
 import sys  
 import time  
 
 while True:  
   sys.stderr.write('Error\n')  
   sys.stdout.write('All Good\n')  
   time.sleep(1)  
``` 

Save this to a file `logging-01.py` (or you can get it from [here](logging-01.py) and then run:

    $ docker run --name logging-01 -ti -d -v $(pwd):/tmp  -w /tmp python:2.7 python logging-01.py
    $ docker logs -f logging-01
    
which will output:

```bash
Error
All Good
Error
All Good
Error
All Good
Error
````

This clearly illustrates how output from `stdout/stderr` gets logged and it's actually available with the `docker logs` command. But wait, isn't this the `json-file` driver? So where is this so called JSON file? Let's inspect our container and see what we can find:

    $ docker inspect logging-01 | grep LogPath
        "LogPath": "/mnt/sda1/var/lib/docker/containers/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1-json.log",
        
your will get a different ouput from the command above but the important thing here is that we get a path to the JSON file where our log messages are stored. So let's take a look at this file:

    tail -2 /mnt/sda1/var/lib/docker/containers/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1/ae1629aceb0b82da6451a981d073243bf7374c07634a377c64a9a7fcea2b40e1-json.log
    {"log":"Error\r\n","stream":"stdout","time":"2015-09-13T03:36:30.120234597Z"}
    {"log":"All Good\r\n","stream":"stdout","time":"2015-09-13T03:36:30.120475567Z"}
    
which is self-explanatory I think ;). 





