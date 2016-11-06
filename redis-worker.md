

```python
from redis import Redis
import time

class RedisQueue(object):
    def __init__(self, key='queue', **redis_kwargs):
       self.__db = Redis(**redis_kwargs)
       self.key = key

    def qsize(self):  # size of the queue
        return self.__db.llen(self.key)

    def isempty(self):
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

def method_that_takes_awhile(foo):
    time.sleep(10)
    print('done with task {}'.format(foo))
```


```python
rq = RedisQueue()

while not rq.isempty():
    task = rq.get()
    method_that_takes_awhile(task)
print('All done')
```
