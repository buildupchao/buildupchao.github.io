---
title: Hive集群合并之应用端的负载均衡算法
date: 2019-05-12 20:46:59
tags: ['Hive', '大数据', '负载均衡算法']
---

![Hive集群合并](https://github.com/buildupchao/ImgStore/blob/master/blog/Hive%E9%9B%86%E7%BE%A4%E5%90%88%E5%B9%B6.png?raw=true)
<!-- more -->
## **0.背景**
有这么一个场景，我们有两个Hive集群，Hive集群1（后面成为1号集群）是一直专享于数据计算平台的，而Hive集群2（后面成为2号集群）是用于其他团队使用的，比如特征，广告等。而由此存在两个主要问题：a) 两个Hive集群共享了同一份MetaData，导致经常会出现在HUE（建立与2号集群上）上建表成功后，但是在计算平台上却无法查询到新建表信息；b) 让运维同学们同时维护两套集群，管理和资源分配调整起来的确是麻烦很多，毕竟也不利于资源的弹性分配。那么鉴于此，经过讨论，需要做这么一样工作：两个集群合二为一，由1号集群合并到2号集群上来。

## **1.集群合并前的思考与分析**
但是，集群合并是不可能一下子全部合并，需要逐步迁移合并（比如每次20个结点）到2号集群。但是这样存在一个问题，计算平台每天使用的计算资源是差不多固定的，而在迁移过程中，1号集群的资源在逐渐减少，显然是不满足计算需求的，所以我们也需要由得到迁移资源的2号集群分担一些压力。<strong>那么重点来了，这就需要我们任务调度器合理的分配任务到1号集群以及2号集群的某个队列。</strong>其实，所谓的任务分配也就是一种负载均衡算法，即任务来了，通过负载均衡算法调度到哪个集群去执行，但是使用哪种负载均衡算法就需要好好探究一下。

### **1.1负载均衡算法的选择**

Q：常用的负载均衡算法有哪些呢？
A：随机算法，轮询，hash算法，加权随机算法，加权轮询算法，一致性hash算法。

- 随机算法
该算法通过产生随机数的方式进行负载，可能会导致任务倾斜，比如大量任务调度到了1好集群，显然不可取，pass。

- 轮询
该算法是通过一个接一个循环往复的方式进行调度，会保证任务分配很均衡，但是我们的1号集群资源是在不断减少的，2号集群资源是在不断增加的，如果均衡分配计算任务，显然也是不合理的，pass。

- hash算法
该算法是基于当前结点的ip的hashCode值来进行调度，那么只要结点ip不变，那么hashCode值就不会变，所有的任务岂不是都提交到一个结点了吗？不合理，pass。

- 加权随机算法
同随机算法，只不过是对每个结点增加了权重，但是因为是随机调度，不可控的，直接pass。

- 加权轮询算法
上面说到，轮询算法可以保证任务分配很均衡，但是无法保证随集群资源的调整进行任务分配的动态调整。此时，如果我们可以依次根据集群迁移情况，设置1号集群与2号集群的任务比重为：7:5 -> 3:2 -> 2:3 -> 完整切换。可行。

- 一致性hash算法
该算法较为复杂，鉴于我们是为了进行集群合并以及保证任务尽量根据集群资源的调整进行合理调度，无需设计太复杂的算法进行处理，故也pass。

## **2.负载均衡算法的落地实现**

虽然我们最终方法选定为<strong>加权轮询算法</strong>，但是它起源于<strong>轮询算法</strong>，那么我们就从<strong>轮询算法</strong>说起。

首选，我们会有Hive集群对应的HS2的ip地址列表，然后我们通过某种算法（这里指的就是负载均衡算法），获取其中一个HS2的ip地址进行任务提交（这就是任务调度）。

### **2.1轮询算法的实现**

我们先定义一个算法抽象接口，它只有一个select方法。

```Java
import java.util.List;

/**
 * @author buildupchao
 * @date: 2019/5/12 21:51
 * @since JDK 1.8
 */
public interface ClusterStrategy {

    /**
     * 负载均衡算法接口
     * @param ipList ip地址列表
     * @return 通过负载均衡算法选中的ip地址
     */
    String select(List<String> ipList);
}
```

轮询算法实现：
```Java
import org.apache.commons.lang3.StringUtils;

import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author buildupchao
 * @date: 2019/5/12 21:57
 * @since JDK 1.8
 */
public class PollingClusterStrategyImpl implements ClusterStrategy {

    private AtomicInteger counter = new AtomicInteger(0);

    @Override
    public String select(List<String> ipList) {
        String selectedIp = null;
        try {
            int size = ipList.size();
            if (counter.get() >= size) {
                counter.set(0);
            }
            selectedIp = ipList.get(counter.get());
            counter.incrementAndGet();

        } catch (Exception ex) {
            ex.printStackTrace();
        }
        if (StringUtils.isBlank(selectedIp)) {
            selectedIp = ipList.get(0);
        }
        return selectedIp;
    }

    public static void main(String[] args) {
        List<String> ipList = Arrays.asList("172.31.0.191", "172.31.0.192");
        PollingClusterStrategyImpl strategy = new PollingClusterStrategyImpl();
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        for (int i = 0; i < 100; i++) {
            executorService.execute(() -> {
                System.out.println(Thread.currentThread().getName() + ":" + strategy.select(ipList));
            });
        }
    }
}
```

运行上述代码，你会发现，线程号为奇数的轮询到的是'172.31.0.191'这个ip，偶数是‘172.31.0.192’这个ip。至于打印出来的日志乱序，那是并发打印返回的ip的问题，并不是获取ip进行任务调度的问题。
![轮询算法](https://github.com/buildupchao/ImgStore/blob/master/blog/%E8%BD%AE%E8%AF%A2%E7%AE%97%E6%B3%95.png?raw=true)

### **2.2加权轮询算法的实现**

既然我们已经实现了轮询算法，那加权轮询怎么实现呢？无非是增加结点被轮询到的比例罢了，我们只需要根据指定的权重，进行轮询即可。因为需要有权重等信息，我们需要重新设计接口。

提供一个Bean进行封装ip以及权重等信息：
```Java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

/**
 * @author buildupchao
 *         Date: 2019/2/1 02:52
 * @since JDK 1.8
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProviderService implements Serializable {
    private String ip;
    // the weight of service provider
    private int weight;
}
```

新的负载均衡算法接口：
```Java
import com.buildupchao.zns.client.bean.ProviderService;

import java.util.List;

/**
 * @author buildupchao
 *         Date: 2019/2/1 02:44
 * @since JDK 1.8
 */
public interface ClusterStrategy {

    ProviderService select(List<ProviderService> serviceRoutes);
}
```

加权轮询算法的实现：
```Java
import com.buildupchao.zns.client.bean.ProviderService;
import com.buildupchao.zns.client.cluster.ClusterStrategy;
import com.google.common.collect.Lists;

import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author buildupchao
 *         Date: 2019/2/4 22:39
 * @since JDK 1.8
 */
public class WeightPollingClusterStrategyImpl implements ClusterStrategy {

    private int counter = 0;
    private Lock lock = new ReentrantLock();

    @Override
    public ProviderService select(List<ProviderService> serviceRoutes) {
        ProviderService providerService = null;

        try {
            lock.tryLock(10, TimeUnit.SECONDS);
            List<ProviderService> providerServices = Lists.newArrayList();
            for (ProviderService serviceRoute : serviceRoutes) {
                int weight = serviceRoute.getWeight();
                for (int i = 0; i < weight; i++) {
                    providerServices.add(serviceRoute);
                }
            }

            if (counter >= providerServices.size()) {
                counter = 0;
            }
            providerService = providerServices.get(counter);
            counter++;
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        } finally {
            lock.unlock();
        }

        if (providerService == null) {
            providerService = serviceRoutes.get(0);
        }
        return providerService;
    }
}
```
你会发现这里的算法实现中不再是通过AtomicInteger来做计数器了，而是借助于```private int counter = 0;```同时借助于```Lock```锁的技术保证计数器的安全访问。只是写法的不同，不用纠结啦！

这样，我们就可以应用这个```加权轮询算法```到我们的任务调度器中了，快速配合运维完成集群迁移合并工作吧！

## **3.总结**
- 常用的负载均衡算法有：随机算法，轮询，hash算法，加权随机算法，加权轮询算法，一致性hash算法
- 和业务场景最契合的负载均衡算法才是最合适的
- 加权轮询负载均衡算法只是在轮询算法基础上根据权重把对应的信息进行平铺多份，从而提高比重实现加权的效果

## **资源链接**

- [负载均衡算法的实现可以参考<strong>zns</strong>项目中的运用](https://github.com/buildupchao/zns/tree/dev-1.0/zns-client/src/main/java/com/buildupchao/zns/client/cluster)
