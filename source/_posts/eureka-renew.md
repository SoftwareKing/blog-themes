---
layout:     post
title:      Spring Cloud Eureka服务续约(Renew)源码分析
date:       2016-11-13 14:00:00 +0800
summary:    Spring Cloud Eureka服务续约源码分析
toc: true
categories:
- Spring Cloud Eureka

tags:
- Spring Cloud Eureka
- Spring Cloud 源码分析
---
**摘要**:在本篇文章中主要对Eureka的Renew(服务续约)，从服务提供者发起续约请求开始分析，通过阅读源码和画时序图的方式，展示Eureka服务续约的整个生命周期。服务续约主要是把服务续约的信息更新到自身的Eureka Server中，然后再同步到其它Eureka Server中。

## Renew(服务续约)
### 概述
Renew（服务续约）操作由Service Provider定期调用，类似于heartbeat。目的是隔一段时间Service Provider调用接口，告诉Eureka Server它还活着没挂，不要把它T了。通俗的说就是它们两之间的心跳检测，避免服务提供者被剔除掉。
请参考:[Spring Cloud Eureka名词解释](http://blog.xujin.org/2016/10/25/spring-cloud-eureka-mid/#名词解释)
<!--more -->
### 服务续约配置
  Renew操作会在Service Provider定时发起，用来通知Eureka Server自己还活着。 这里有两个比较重要的配置需要如下，可以在Run之前配置。
```java
  eureka.instance.leaseRenewalIntervalInSeconds
```
  Renew频率。默认是`30秒`，也就是每30秒会向Eureka Server发起Renew操作。
```java
  eureka.instance.leaseExpirationDurationInSeconds
```
 服务失效时间。默认是`90秒`，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把`Service Provider剔除`。
## Renew源码分析
### 服务提供者实现细节
 服务提供者发发起服务续约的时序图，如下图所示,大家先直观的看一下时序图，等阅读完源码再回顾一下。
![服务提供者发起续约时序图](/images/spring-cloud-netflix/eureka/service-renew.png)

1. 在com.netflix.discovery.DiscoveryClient.initScheduledTasks()中的1272行，TimedSupervisorTask会定时发起服务续约，代码如下所示:
```java
 // Heartbeat timer
   scheduler.schedule(
      new TimedSupervisorTask(
             "heartbeat",
              scheduler,
              heartbeatExecutor,
              renewalIntervalInSecs,
              TimeUnit.SECONDS,
              expBackOffBound,
               new HeartbeatThread()
             ),
   renewalIntervalInSecs, TimeUnit.SECONDS);
```
2.在com.netflix.discovery.DiscoveryClient中的1393行，有一个`HeartbeatThread`线程发起续约操作
```java
 private class HeartbeatThread implements Runnable {

        public void run() {
            //调用eureka-client中的renew
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
}
```
renew()调用eureka-client-1.4.11.jarcom.netflix.discovery.DiscoveryClient中`829`行renew()发起`PUT Reset`请求，调用com.netflix.eureka.resources.InstanceResource中的renewLease()续约。
```java
 /**
     * Renew with the eureka service by making the appropriate REST call
     */
    boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug("{} - Heartbeat status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == 404) {
                REREGISTER_COUNTER.increment();
                logger.info("{} - Re-registering apps/{}", PREFIX + appPathIdentifier, instanceInfo.getAppName());
                return register();
            }
            return httpResponse.getStatusCode() == 200;
        } catch (Throwable e) {
            logger.error("{} - was unable to send heartbeat!", PREFIX + appPathIdentifier, e);
            return false;
        }
  }
```
### Netflix中的Eureka Core实现细节
   NetFlix中Eureka Core中的服务续约时序图，如下图所示。
  ![服务续约时序图](/images/spring-cloud-netflix/eureka/eureka-renew.png)
 1. 打开`com.netflix.eureka.resources.InstanceResource`中的`106`行的`renewLease()`方法，代码如下:
```java
    private final PeerAwareInstanceRegistry registry
    @PUT
    public Response renewLease(
            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
            @QueryParam("overriddenstatus") String overriddenStatus,
            @QueryParam("status") String status,
            @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
        boolean isFromReplicaNode = "true".equals(isReplication);
        //调用
        boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);
        //其余省略
    }

```
 2. 点开registry.renew(app.getName(), id, isFromReplicaNode);我们可以看到，调用了`org.springframework.cloud.netflix.eureka.server.InstanceRegistry`中的`renew（）`方法，代码如下:
```java
    @Override
	public boolean renew(final String appName, final String serverId,
			boolean isReplication) {
		log("renew " + appName + " serverId " + serverId + ", isReplication {}"
				+ isReplication);
		List<Application> applications = getSortedApplications();
		for (Application input : applications) {
			if (input.getName().equals(appName)) {
				InstanceInfo instance = null;
				for (InstanceInfo info : input.getInstances()) {
					if (info.getHostName().equals(serverId)) {
						instance = info;
						break;
					}
				}
				publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
						instance, isReplication));
				break;
			}
		}
        //调用com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl中的renew方法
		return super.renew(appName, serverId, isReplication);
}

```
3.从`super.renew()`看到调用了父类中的`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl`中`420`行的`renew()`方法，代码如下:
```java
 public boolean renew(final String appName, final String id, final boolean isReplication) {
        //服务续约成功，
        if (super.renew(appName, id, isReplication)) {
            //然后replicateToPeers同步其它Eureka Server中的数据
            replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
            return true;
        }
        return false;
}
```
3.1 从上面代码中`super.renew(appName, id, isReplication)`可以看出调用的是com.netflix.eureka.registry.AbstractInstanceRegistry中`345`行的renew()方法，代码如下所示
```java
  public boolean renew(String appName, String id, boolean isReplication) {
        RENEW.increment(isReplication);
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToRenew = null;
        if (gMap != null) {
            leaseToRenew = gMap.get(id);
        }
        if (leaseToRenew == null) {
            RENEW_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
            return false;
        } else {
            InstanceInfo instanceInfo = leaseToRenew.getHolder();
            if (instanceInfo != null) {
                // touchASGCache(instanceInfo.getASGName());
                InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                        instanceInfo, leaseToRenew, isReplication);
                if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                    logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                            + "; re-register required", instanceInfo.getId());
                    RENEW_NOT_FOUND.increment(isReplication);
                    return false;
                }
                if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                    Object[] args = {
                            instanceInfo.getStatus().name(),
                            instanceInfo.getOverriddenStatus().name(),
                            instanceInfo.getId()
                    };
                    logger.info(
                            "The instance status {} is different from overridden instance status {} for instance {}. "
                                    + "Hence setting the status to overridden status", args);
                    instanceInfo.setStatus(overriddenInstanceStatus);
                }
            }
            renewsLastMin.increment();
            leaseToRenew.renew();
            return true;
        }
    }
```
其中 `leaseToRenew.renew()`是调用com.netflix.eureka.lease.Lease<T>中的`62`行的renew()方法
```java
    /**
     * Renew the lease, use renewal duration if it was specified by the
     * associated {@link T} during registration, otherwise default duration is
     * {@link #DEFAULT_DURATION_IN_SECS}.
     */
    public void renew() {
        lastUpdateTimestamp = System.currentTimeMillis() + duration;

    }
```
3.2 replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);调用自身的`replicateToPeers()`方法，在com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl中的`618`行，主要接口实现方式和register基本一致：首先更新自身Eureka Server中服务的状态，再同步到其它Eureka Server中。
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
            // 同步把续约信息同步到其它的Eureka Server中
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


