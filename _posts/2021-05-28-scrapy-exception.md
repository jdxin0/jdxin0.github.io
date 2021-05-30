---
layout: post
title:  "Scrapy异常处理"
date:   2021-05-28 15:11:14 +0800
categories: 
---

假设需求是这样的，请求一个[`json`接口](https://httpbin.org/ip)，将其转化为`dict`，然后调用下面的处理函数

```python
def handle(data):
    pass
```

- 首先使用`requests`实现一个

```python
import requests

if __name__ == '__main__':
    r = requests.get('https://httpbin.org/ip')
    handle(r.json())
```

- 一个工业级的`scrapy`爬虫会写成这个样子, 省略了异常处理细节(直接`pass`)以及日志处理，命令行解析和速率控制等

```python
import json
import scrapy
from scrapy.crawler import CrawlerProcess

class TestSpider(scrapy.Spider):
    name = 'test'

    def start_requests(self):
        yield scrapy.Request(
            'https://httpbin.org/ip',
            meta={
                'dont_redirect': True,
                'handle_httpstatus_list': [302, 404]
            },
            headers={
                'referer': 'https://httpbin.org', 
                'user-agent': 'fake user agent'
            },
            priority=1,
            dont_filter=True,
            callback=self.parse,
            errback=self.error_back,
            cookies={'cookie1': 'cookie1_value'},
        )

    def parse(self, response):
        if response.status == 302:
            if isinstance(response.headers['Location'], list):
                redirect_url = response.headers['Location'][0].decode('utf8')
            else:
                redirect_url = response.headers['Location'].decode('utf8')

            if redirect_url.contains('login'):
                # need login, maybe cookies expire
                pass
            elif if redirect_url.contains('nojson'):
                # page doesn't exists, like 404
                pass
            else:
                # other possibilities
                pass
        elif esponse.status == 404:
            pass
        else:
            try:
                source = response.body.decode('utf-8')
            except UnicodeDecodeError:
                # page doesn't seem like utf-8 encoding
                pass
            else:
                try:
                    page = json.loads(source)
                except json.decoder.JSONDecodeError:
                    # maybe it's not a json after all
                else:
                    handle(data)

    def error_back(self, failure):
        # three types of error
        # 1. max retry times reached by RETRY_HTTP_CODES
        # 2. connection error, like timeout, connection refused 
        # 3. error code not in RETRY_HTTP_CODES in settings 
        # or handle_httpstatus_list in request meta
        pass

if __name__ == '__main__':
    settings = {'RETRY_HTTP_CODES': [500, 502, 503, 504, 408]}
    process = CrawlerProcess(settings)
    process.crawl(TestSpider)
    process.start()
```

现实往往过于残酷…

省略的东西是都要加上的