
## Introduction to Redis

Redis is a simple, in-memory key-value store. It has many useful applications when used in conjunction with Python, R, or similar languages where you want an extra layer of data persistence and/or parallel processing.

Best of all, it's super easy to pick up and add to your arsenal.

* The docs are really good: http://redis.io/commands
* A useful primer: http://openmymind.net/redis.pdf
* A book co-written by my ex-coworker at Yipit: https://books.google.com/books/about/Redis_Essentials.html?id=54REjgEACAAJ
* Redis for R: https://cran.r-project.org/web/packages/rredis/vignettes/rredis.pdf


```python
from redis import Redis
import pandas as pd
import numpy as np

# without decode_responses all responses will come to you as bytes
r = Redis(host='localhost', port=6379, db=0, decode_responses=True)
```

### Data types
Redis supports 5 data types, each with their specific syntaxes (there's a lot in common):
* strings
* hashes
* lists
* sets
* sorted sets

Let's take a look these in the context of `redis-python`, a Python wrapper for Redis.

#### Strings
* The most basic type; a simple key-value pair
* Python equivalent: Strings


```python
r.set('AAPL-rec', 'buy')
r.get('AAPL-rec')
```


```python
r.append('AAPL-rec', '-strong')
r.get('AAPL-rec')
```


```python
r.set('AAPL-conviction', 5)
r.incrby('AAPL-conviction', 2)
r.get('AAPL-conviction')
```

#### Hashes
* Each hash has a set of key-value pairs
* Python equivalent: Dicts


```python
r.hset('AAPL-data', 'last-price', 114.06)
r.hget('AAPL-data', 'last-price')
```


```python
aapl_data = {
    'pe': 13.41,
    'eps': 8.25,
    'mkt_cap': 615360.2,
    'headquarters': 'Cupertino, CA',
    'employees': 100000
}
r.hmset('AAPL-data', aapl_data)
r.hgetall('AAPL-data')  # note how last-price also persisted under the key AAPL-data
```

#### Lists
* Lists let you store and manipulate an array of values for a given key
* Redis lists are all LIFO, and inserts and pops are from the "head" of the list
* Python equivalent: Lists


```python
users = ['alice', 'bob', 'cathy', 'david']
for user in users:
    r.lpush('users', user)
r.lpop('users')
```


```python
r.lindex('users', 0)
```


```python
r.ltrim('users', 0, 2)  # inclusive

r.linsert('users', 'before', 'alice', 'max')  # rare case where it looks up values
r.lrange('users', 0, 10)
```

#### Sets
* Sets are used to store unique values and provide a number of set-based operations, like unions
* Python equivalent: Sets



```python
r.sadd('tickers', 'AAPL', 'MCD', 'CMG', 'WMT')
r.sismember('tickers', 'AAPL')
```


```python
r.sadd('more-tickers', 'BBY', 'CAKE', 'DRI')
r.sunionstore('tickers', 'tickers', 'more-tickers')
r.smembers('tickers')
```

#### Sorted Sets
* Sorted sets are sets, but each element is linked to a score
* Python equivalent: Dicts (with numerical values)


```python
r.zadd('portfolio', 'AAPL', 50, 'MCD', 20, 'CMG', -50, 'WMT', -20)
r.zcard('portfolio')
```


```python
r.zincrby('portfolio', 'CMG', -25)
r.zscore('portfolio', 'CMG')
```


```python
r.zrange('portfolio', 0, 100, withscores=True)
```


```python
r.zrevrank('portfolio', 'CMG')
```

#### Other useful commands

* keys() shows all the keys in the current database
* delete(key) deletes a key
* flushdb() deletes all keys in the database
* flushall() deletes all keys in all db's - use this carefully! You may have forgotten about keys in other db's
* rename(key, newkey) renames key to newkey
* sort() sorts a list, set or sorted set
* setex(), expire() sets expiration flag on a key for n seconds, after which the key will disappear
* pubsub() creates a publisher-subscriber pattern
* eval() evaluates Lua scripts


```python
print(r.sort('users', desc=True, alpha=True))
```


```python
r.keys()
```


```python
r.flushdb()
r.keys()
```


```python
r.set('key', 'value')
r.expire('key', 100)
```


```python
r.get('key')
```

### Redis applications

How will this save me time / make my job easier? Here are two generic examples.

#### Example 1: data persistence

Redis is great for persisting objects across your programming instances.


```python
from redis import Redis
import random
from string import ascii_letters

r = Redis(host='localhost', port=6379, db=0, decode_responses=True)

hugestring = ''.join([random.choice(ascii_letters) for i in range(1000000)])
```


```python
hugestring  # Compare how long it takes for jupyter nb to display it...
```


```python
r.set('key', hugestring)  # ...with how long it took for it to be stored in Redis
```

Have a general python object that you want to store in Redis? No problem - just `pickle` it


```python
import pickle
import requests

res = requests.get('http://www.google.com')  # this is a requests.Response object
pickled_object = pickle.dumps(res)

r.set('foo', pickled_object)
```

### Example 2: task queues

Redis's `list` data type is a natural choice for task queues. This is a common Redis application for a language with limited async capabilities like Python. There are many Redis-based wrappers like [rq](http://python-rq.org/) and [celery](http://www.celeryproject.org/), etc but here's a low level implementation:


```python
# this should be a standalone module, but I've shown the code for sake of illustration
from redis import Redis

class RedisQueue(object):
    def __init__(self, key='queue', **redis_kwargs):
       self.__db = Redis(**redis_kwargs)
       self.key = key

    def queue_size(self):
        return self.__db.llen(self.key)

    def is_empty(self):
        return self.qsize() == 0

    def put(self, item):
        self.__db.rpush(self.key, item)

    def get(self, block=True, timeout=None):
        if block:
            item = self.__db.blpop(self.key, timeout=timeout)
        else:
            item = self.__db.lpop(self.key)
        if item:
            item = item[1]
        return item
```


```python
import time

def method_that_takes_awhile(foo):
    time.sleep(5)
    print('done with task {}'.format(foo))

tasks = ['A', 'B', 'C', 'D', 'E', 'F', 'G']
```


```python
# the way most of us probably do it
for task in tasks:
    method_that_takes_awhile(task)
print('all done')
```


```python
# put the tasks in the queue instead

rq = RedisQueue()
for task in tasks:
    rq.put(task)
```


```python
while not rq.is_empty():
    task = rq.get()
    method_that_takes_awhile(task)
print('All done')
```

### Key redis drawbacks

* Unable to search by values. Redis isn't sql!
* You cannot roll back transactions (be careful with `flushdb` and `flushall`...)
* Redis resides RAM so may be costly for your local machine
    * Solution: dedicated machine that runs Redis. This was our setup at Yipit
* Key management may be cumbersome if you don't remember what's in what key
