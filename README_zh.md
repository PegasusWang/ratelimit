# 基于redis的分布式限频库
![travis-ci](https://www.travis-ci.org/vearne/ratelimit.svg?branch=master)

---

## 总述
这个库的目标是为了能够简单粗暴的实现分布式限频功能, 类似于ID生成器的用法，
client每次从Redis拿回-批数据(在这里只是一个数值)进行消费，
只要没有消费完，就没有达到频率限制。

* [English README](https://github.com/vearne/ratelimit/blob/master/README.md)

## 优势
* 依赖少，只依赖redis，不需要专门的服务
* 使用的redis自己的时钟，不需要相应的服务器时钟完全一致
* 线程(协程)安全
* 系统开销小，对redis的压力很小

## 安装
```
go get github.com/vearne/ratelimit
```
## 用法
### 1. 创建 redis.Client
依赖 "github.com/go-redis/redis"    
同时支持redis 主从模式和cluster模式
```
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "xxx", // no password set
		DB:       0,  // use default DB
	})
```
```
	client := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs:    []string{"127.0.0.1:6379"},
		Password: "xxxx",
	})
```


### 2. 创建限频器
```
    limiter, err := ratelimit.NewTokenBucketRateLimiter(client, 
            "push", time.Second, 200, 20, 5)
```
表示允许每秒操作200次
```
	limiter, err := ratelimit.NewTokenBucketRateLimiter(client, 
	        "push", time.Minute, 200, 20, 5)
```
表示允许每分钟操作200次

支持多种算法
#### 2.1 计数器算法
```
func NewCounterRateLimiter(client redis.Cmdable, key string, duration time.Duration,
	throughput int,
	batchSize int) (Limiter, error)
```
|参数|说明|
|:---|:---|
|key|redis中key|
|duration|表明在duration时间间隔内允许操作throughput次|
|throughput|表明在duration时间间隔内允许操作throughput次|
|batchSize|每次从redis拿回的可用操作的数量|

#### 2.2 令牌桶算法
```
func NewTokenBucketRateLimiter(client redis.Cmdable, key string, duration time.Duration,
	throughput int, maxCapacity int,
	batchSize int) (Limiter, error)
```
|参数|说明|
|:---|:---|
|key|redis中key|
|duration|表明在duration时间间隔内允许操作throughput次|
|throughput|表明在duration时间间隔内允许操作throughput次|
|maxCapacity|令牌桶最多可保存的令牌数|
|batchSize|每次从redis拿回的可用操作的数量|

#### 2.3 漏桶算法
```
func NewLeakyBucketLimiter(client redis.Cmdable, key string, duration time.Duration,
	throughput int) (Limiter, error)
```

|参数|说明|
|:---|:---|
|key|redis中key|
|duration|表明在duration时间间隔内允许操作throughput次|
|throughput|表明在duration时间间隔内允许操作throughput次|


完整例子
```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"github.com/vearne/ratelimit"
	"math/rand"
	"sync"
	"time"
)

func consume(r ratelimit.Limiter, group *sync.WaitGroup) {
	for {
		if r.Take() {
			group.Done()
			fmt.Println("curr", time.Now())
		} else {
			time.Sleep(time.Duration(rand.Intn(10)+1) * time.Millisecond)
		}
	}
}

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "xxeQl*@nFE", // password set
		DB:       0,       // use default DB
	})


	limiter, err := ratelimit.NewTokenBucketRateLimiter(client, "push",
		time.Second,
		10,
		5,
		2,)

	if err!= nil{
		fmt.Println("error", err)
		return
	}

	var wg sync.WaitGroup
	total := 500
	wg.Add(total)
	start := time.Now()
	for i := 0; i < 100; i++ {
		go consume(limiter, &wg)
	}
	wg.Wait()
	cost := time.Since(start)
	fmt.Println("cost", time.Since(start), "rate", float64(total)/cost.Seconds())
}
```
### 依赖
[go-redis/redis](https://github.com/go-redis/redis)

### 致谢
模块的开发受到了资料1的启发，在此表示感谢



### 资料
1. [性能百万/s：腾讯轻量级全局流控方案详解](http://wetest.qq.com/lab/view/320.html)

### 感谢
[![jetbrains](img/jetbrains.svg)](https://www.jetbrains.com/?from=vearne)

