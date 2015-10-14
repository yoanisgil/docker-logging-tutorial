# Logging with Docker - Part 1.1

So it's been a while since [I started this blog series on logging with Docker](https://medium.com/@yoanis_gil/logging-with-docker-part-1-b23ef1443aac) and though I said the next article will be about *gelf* and *fluentd*, I want to take the time to provide more realistic examples showing what it takes to integrate logging in your application with Docker. Because let's face it, this:

```python
import sys
import time

while True:
    sys.stderr.write('Error\n')
    sys.stdout.write('All Good\n')
    time.sleep(1)
```

hardly counts as an application, nor you get paid for writing such code ;).