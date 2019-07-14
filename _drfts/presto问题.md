观察到两个影响
- 日志报错
```
2019-07-10T18:57:37.737+0800	ERROR	Discovery-0	io.airlift.discovery.client.CachingServiceSelector	Cannot connect to discovery server for refresh (presto/general): Lookup of presto failed with status code 404

2019-07-10T18:59:29.790+0800	ERROR	Announcer-1	io.airlift.discovery.client.Announcer	Service announcement failed after 838.04us. Next request will happen within 1000.00ms
2019-07-10T18:59:30.791+0800	ERROR	Announcer-2	io.airlift.discovery.client.Announcer	Service announcement failed after 701.55us. Next request will happen within 1000.00ms
2019-07-10T18:59:31.792+0800	ERROR	Announcer-3	io.airlift.discovery.client.Announcer	Service announcement failed after 666.06us. Next request will happen within 1000.00ms
```

- 接口不能访问
http://172.22.11.50:9080/v1/service  报404
