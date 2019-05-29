---
title: 使用flask,uwsgi和nginx搭建高并发http server
date: 2019-05-29 10:30:00
categories: web
tags:
  - flask uwsgi nginx
---
## environment
Ubuntu 16.04.6 LTS
Flask 1.0.3
uwsgi 2.0.18
nginx 1.10.3

python 3.6

## install

```
pip install flask
pip install uwsgi
sudo apt-get install nginx
```

安装uwsgi时可能会有gcc版本过高导致安装失败的问题，需要安装低版本gcc

```
sudo apt-get install gcc-4.7
sudo rm /usr/bin/gcc
sudo ln -s /usr/bin/gcc-4.7 /usr/bin/gcc
```

## config

### 使用flask自带的wsgi服务器运行

app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
		return 'hello'
		
if __name__ = '__main__':
		app.run(host='0.0.0.0', port=6600)
```

运行：

```python
python app.py
```

访问：

```
http://ip:6600/
```



### 使用uwsgi服务器加载flask应用

run.py

```python
from app import app
```

uwsgi.ini

```
[uwsgi]
master = false
http = 127.0.0.1:6602
wsgi-file = run.py
callable = app
processes = 4
threads = 2
socket = /tmp/hmeplus.sock
chmod-socket = 666
#logto = uwsgi.log
```

运行：

```
uwsgi uwsgi.ini
```

访问：

```
http://ip:6602/
```

**注意**：

1. 如果程序中有起自己的线程，并与主线程共享变量，请将ini配置文件中master置为false，否则会产生fork操作，导致进程间数据不同步。具体可以参考：<https://stackoverflow.com/questions/32059634/python3-threading-with-uwsgi>

2. 如果需要将uwsgi的打印消息导入日志，可以开启logto选项。



### 使用Nginx

创建新的配置文件/etc/nginx/sites-available/uwsgi

```
server {
        listen 6601;
        server_name _;

        location / {
                include uwsgi_params;
                uwsgi_pass unix:///tmp/hmeplus.sock;
        }
}
```

创建软链接

```shell
sudo ln -s /etc/nginx/sites-available/uwsgi /etc/nginx/sites-enabled
```

检查配置文件语法

```
sudo nginx -t
```

重启nginx服务

```
sudo systemctl restart nginx
```

访问:

```
http://ip:6601/
```

## benckmark

对比测试flask+uwsgi+nginx与http.server(python3内置)并发性能。使用Apache Benchmark（AB）工具。

```
ab -n 200 -c 200 -T "application/x-www-form-urlencoded" -p post.txt http://ip:port/
```

### nginx result

| Concurrency   Level:      200                                |
| ------------------------------------------------------------ |
| Time taken for tests:   2.899 seconds                        |
| Complete requests:      200                                  |
| Failed requests:        0                                    |
| Total transferred:      33000 bytes                          |
| Total body sent:        12512600                             |
| HTML transferred:       0 bytes                              |
| Requests per second:    68.99 [#/sec] (mean)                 |
| Time per request:       2898.987 [ms] (mean)                 |
| Time per request:       14.495 [ms] (mean, across all   concurrent requests) |
| Transfer rate:          11.12 [Kbytes/sec] received          |
| 4215.04 kb/s sent                                            |
| 4226.15 kb/s total                                           |

### http.server result

| Concurrency   Level:      200                                |
| ------------------------------------------------------------ |
| Time taken for tests:   51.808 seconds                       |
| Complete requests:      200                                  |
| Failed requests:        0                                    |
| Total transferred:      0 bytes                              |
| Total body sent:        12512600                             |
| HTML transferred:       0 bytes                              |
| Requests per second:    3.86 [#/sec] (mean)                  |
| Time per request:       51807.559 [ms] (mean)                |
| Time per request:       259.038 [ms] (mean, across all   concurrent requests) |
| Transfer rate:          0.00 [Kbytes/sec] received           |
| 235.86 kb/s sent                                             |
| 235.86 kb/s total                                            |

