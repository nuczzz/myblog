### go-micro RequestTimeout

#### 背景
公司项目采用go-micro微服务框架，最近发现微服务之间调用偶尔会出现如下错误：
```
{"id":"go.micro.client","code":408,"detail":"call timeout: context deadline exceeded","status":"Request Timeout"}
```
初步查看原因，是与微服务调用的context有关，相关代码如下：
```
//toto
```

//todo
