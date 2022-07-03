---
title: "我是生活二"
date: 2020-08-02T22:41:23+08:00
draft: false
categories: 生活
tags: ["随想"]
plocation: top_right_four
pbackground: https://static.is26.com/uploads/2019/07/cat-travel.jpg
---

> 辛苦移步微信公众号点赞、在看、关注：
[分布式锁 | 电商防超卖的N+1个坑！](https://mp.weixin.qq.com/s/82S1AbBeJzYuGhl6FvjQcg)  

   大家好，我是西瓜。  
  今天和同事讨论库存防超卖问题，发现虽然只是简单的库存扣减场景，却隐藏着很多坑，一不小心就容易翻车，让西瓜推土机来填平这些坑。

## 单实例环境
 一般电商体系防止库存超卖，主要有以下几种方式：
![导图](http://file-ok.hanyu.cool/hysite/hysite-tech/article/image/2021-03-23/oversold.png)
防止库存超卖，最先想到的可能就是「锁」，如果是一些单实例部署的库存服务，大部分情况下我们可以使用以下锁或并发工具类：
![单机锁](http://file-ok.hanyu.cool/hysite/hysite-tech/article/image/2021-03-23/java_lock.png)
这三个任何一个都可以保证同一单位时间只有一个线程能够进行库存扣减，废话不多说，上码！
```java
/**
     * 库存扣减（伪代码 ReentrantLock )
     * @param stockRequestDTO
     * @return Boolean
     */
    public Boolean stockHandle(StockRequestDTO stockRequestDTO) {
        // 日志打印...校验...前置处理等...
        int stock = stockMapper.getStock(stockRequestDTO.getGoodsId());
        reentrantLock.lock();
        try {
            int result = stock > 0 ? 
                    stockMapper.updateStock(stockRequestDTO.getGoodsId(), --stock) : 0;
            return result > 0 ? true : false;
        } catch (SQLException e) {
            // 异常日志打印及处理...
            return false;
        } finally {
            reentrantLock.unlock();
        }
    }
    
   /**
     * 库存扣减（伪代码 synchronized )
     * @param stockRequestDTO
     * @return Boolean
     */
    public synchronized Boolean stockHandle(StockRequestDTO stockRequestDTO){
        // 执行业务逻辑...
    }

    /**
     * 库存扣减（伪代码 Semaphore )
     * @param stockRequestDTO
     * @return Boolean
     */
    public Boolean stockHandle(StockRequestDTO stockRequestDTO) {
      try{
          semaphore.acquire();
          // 执行业务逻辑...
      } finally {
          semaphore.release();
        }
    }
```

如果你的项目是单实例部署，那么使用以上锁或并发工具中的一种，都可以有效的防止超卖出现。

## 分布式环境
  但现在的互联网公司，基本都是负载均衡的方式，访问集群中多个实例的，所以基于JVM级别的锁无法发挥作用，需要引入第三方组件来解决，分布式锁登场。

  如果想实现分布式环境下的锁机制，最简单的莫过于利用MySQL的锁机制:
  ![DBLOCK](http://file-ok.hanyu.cool/hysite/hysite-tech/article/image/2021-03-23/db_lock.png)
  
```sql
*** 使用悲观锁实现 *** 
