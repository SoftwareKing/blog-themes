---
layout:     post
title:      Spring Cloud Eureka服务下线(Cancel)源码分析
date:       2016-11-19 14:00:00 +0800
summary:    Spring Cloud Eureka服务下线源码分析
toc: true
categories:
- Spring Cloud Eureka

tags:
- Spring Cloud Eureka
- Spring Cloud 源码分析
---
**摘要**:在本篇文章中主要对Eureka的Cancel(服务下线)进行源码分析，在Service Provider服务shut down的时候，需要及时通知Eureka Server把自己剔除，从而避免其它客户端调用已经下线的服务，导致服务不可用。

## Cancel(服务下线)
### 概述
在Service Provider服务shut down的时候，需要及时通知Eureka Server把自己剔除，从而避免客户端调用已经下线的服务。
## 服务提供者端源码分析
 1. 在eureka-client-1.4.1中的com.netflix.discovery.DiscoveryClient中shutdown()的`867`行。
<!--more-->
```java
 /**
     * Shuts down Eureka Client. Also sends a deregistration request to the
     * eureka server.
     */
    @PreDestroy
    @Override
    public synchronized void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            logger.info("Shutting down DiscoveryClient ...");

            if (statusChangeListener != null && applicationInfoManager != null) {
                applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
            }

            cancelScheduledTasks();

            // If APPINFO was registered
            if (applicationInfoManager != null && clientConfig.shouldRegisterWithEureka()) {
                applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
                //调用下线接口
                unregister();
            }

            if (eurekaTransport != null) {
                eurekaTransport.shutdown();
            }

            heartbeatStalenessMonitor.shutdown();
            registryStalenessMonitor.shutdown();

            logger.info("Completed shut down of DiscoveryClient");
        }
   }
```
>`Tips` `@PreDestroy`注解或`shutdown()`的方法是服务下线的入口

2. 在eureka-client-1.4.1中的`com.netflix.discovery.DiscoveryClient`中`unregister（）`的`897`行
```java
 /**
  * unregister w/ the eureka service.
  */
 void unregister() {
        // It can be null if shouldRegisterWithEureka == false
  if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
    try {
         logger.info("Unregistering ...");
         //发送服务下线请求
         EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
         logger.info(PREFIX + appPathIdentifier + " - deregister  status: " + httpResponse.getStatusCode());
      } catch (Exception e) {
                logger.error(PREFIX + appPathIdentifier + " - de-registration failed" + e.getMessage(), e);
      }
 }
}
```
## Eureka Server服务下线实现细节
1. 在`com.netflix.eureka.resources.InstanceResource`中的`280`行中的`cancelLease()`方法
```java
@DELETE
public Response cancelLease(
 @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
   //调用cancel
   boolean isSuccess = registry.cancel(app.getName(), id,
                "true".equals(isReplication));

  if (isSuccess) {
     logger.debug("Found (Cancel): " + app.getName() + " - " + id);
            return Response.ok().build();
   } else {
     logger.info("Not Found (Cancel): " + app.getName() + " - " + id);
            return Response.status(Status.NOT_FOUND).build();
       }
}
```
2. 在`org.springframework.cloud.netflix.eureka.server.InstanceRegistry`中的`95`行的`cancel()`方法，
```java
@Override
public boolean cancel(String appName, String serverId, boolean isReplication) {
   handleCancelation(appName, serverId, isReplication);
   //调用父类中的cancel
   return super.cancel(appName, serverId, isReplication);
}
```
3. 在`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl`中的`376`行
```java
 @Override
    public boolean cancel(final String appName, final String id,
                          final boolean isReplication) {
     if (super.cancel(appName, id, isReplication)) {
            //服务下线成功后，同步更新信息到其它Eureka Server节点
            replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
            synchronized (lock) {
                if (this.expectedNumberOfRenewsPerMin > 0) {
                    // Since the client wants to cancel it, reduce the threshold (1 for 30 seconds, 2 for a minute)
                    this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin - 2;
                    this.numberOfRenewsPerMinThreshold =
                            (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                }
            }
            return true;
     }
        return false;
}
```
4.在com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl中的`618`行，主要接口实现方式和register基本一致：首先更新自身Eureka Server中服务的状态，再同步到其它Eureka Server中。
```java
private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // If it is a replication already, do not replicate again as this will create a poison replication
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }
            // 同步把服务信息同步到其它的Eureka Server中
            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // If the url represents this host, do not replicate to yourself.
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                //根据action做相应操作的同步
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
 }
```
至此，Eureka服务续约源码分析结束，大家有兴趣可自行阅读。
### 源码分析链接
 其它源码分析链接:
 Spring Cloud中@EnableEurekaClient源码分析:
 http://blog.xujin.org/2016/11/06/enableEurekaClient-annonation/
 Spring Cloud Eureka服务注册源码分析：
 http://blog.xujin.org/2016/11/01/spring-cloud-eureka-register/
 Spring Cloud Eureka服务续约(Renew)源码分析
 http://blog.xujin.org/2016/11/13/eureka-renew/



