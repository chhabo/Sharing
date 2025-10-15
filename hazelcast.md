
通过配置控制 Hazelcast 使用 Multicast 或 Kubernetes 发现机制

在 Spring Boot 应用中，可以通过外部配置灵活控制 Hazelcast 使用 Multicast 或 Kubernetes 服务发现机制。以下是完整的实现方案：

1. 配置属性设计

在 
"application.yml" 中添加配置属性：

hazelcast:
  discovery:
    # 可选值: MULTICAST, KUBERNETES
    type: KUBERNETES
    multicast:
      enabled: false
      group: 224.2.2.3
      port: 54327
    kubernetes:
      enabled: true
      namespace: default
      service-name: hazelcast-service
      resolve-not-ready-addresses: true

2. 配置属性类

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "hazelcast.discovery")
public class HazelcastDiscoveryProperties {

    private DiscoveryType type = DiscoveryType.KUBERNETES;
    private MulticastProperties multicast = new MulticastProperties();
    private KubernetesProperties kubernetes = new KubernetesProperties();

    public enum DiscoveryType {
        MULTICAST, KUBERNETES
    }

    public static class MulticastProperties {
        private boolean enabled = false;
        private String group = "224.2.2.3";
        private int port = 54327;

        // Getters and Setters
    }

    public static class KubernetesProperties {
        private boolean enabled = true;
        private String namespace = "default";
        private String serviceName = "hazelcast-service";
        private boolean resolveNotReadyAddresses = true;

        // Getters and Setters
    }

    // Getters and Setters
}

3. 动态 Hazelcast 配置

import com.hazelcast.config.Config;
import com.hazelcast.config.JoinConfig;
import com.hazelcast.config.NetworkConfig;
import com.hazelcast.config.TcpIpConfig;
import com.hazelcast.kubernetes.HazelcastKubernetesDiscoveryStrategyFactory;
import com.hazelcast.spi.discovery.integration.DiscoveryServiceProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HazelcastDynamicConfig {

    private final HazelcastDiscoveryProperties discoveryProperties;

    @Autowired
    public HazelcastDynamicConfig(HazelcastDiscoveryProperties discoveryProperties) {
        this.discoveryProperties = discoveryProperties;
    }

    @Bean
    public Config hazelcastConfig() {
        Config config = new Config();
        config.setClusterName("dynamic-discovery-cluster");
        
        NetworkConfig networkConfig = config.getNetworkConfig();
        networkConfig.setPort(5701).setPortAutoIncrement(true);
        
        JoinConfig joinConfig = networkConfig.getJoin();
        
        // 禁用所有默认发现机制
        joinConfig.getMulticastConfig().setEnabled(false);
        joinConfig.getTcpIpConfig().setEnabled(false);
        joinConfig.getAwsConfig().setEnabled(false);
        joinConfig.getGcpConfig().setEnabled(false);
        joinConfig.getAzureConfig().setEnabled(false);
        joinConfig.getKubernetesConfig().setEnabled(false);
        
        // 根据配置启用指定的发现机制
        switch (discoveryProperties.getType()) {
            case MULTICAST:
                configureMulticast(joinConfig);
                break;
            case KUBERNETES:
                configureKubernetes(joinConfig);
                break;
            default:
                throw new IllegalArgumentException("Unsupported discovery type: " + discoveryProperties.getType());
        }
        
        return config;
    }

    private void configureMulticast(JoinConfig joinConfig) {
        HazelcastDiscoveryProperties.MulticastProperties multicast = discoveryProperties.getMulticast();
        
        joinConfig.getMulticastConfig()
            .setEnabled(true)
            .setMulticastGroup(multicast.getGroup())
            .setMulticastPort(multicast.getPort());
    }

    private void configureKubernetes(JoinConfig joinConfig) {
        HazelcastDiscoveryProperties.KubernetesProperties kubernetes = discoveryProperties.getKubernetes();
        
        joinConfig.getKubernetesConfig()
            .setEnabled(true)
            .setProperty("namespace", kubernetes.getNamespace())
            .setProperty("service-name", kubernetes.getServiceName())
            .setProperty("resolve-not-ready-addresses", String.valueOf(kubernetes.isResolveNotReadyAddresses()));
    }
}

4. 使用 Spring Profile 进行环境区分

application-dev.yml (开发环境使用 Multicast)

spring:
  profiles: dev

hazelcast:
  discovery:
    type: MULTICAST
    multicast:
      enabled: true
      group: 224.2.2.3
      port: 54327

application-prod.yml (生产环境使用 Kubernetes)

spring:
  profiles: prod

hazelcast:
  discovery:
    type: KUBERNETES
    kubernetes:
      enabled: true
      namespace: production
      service-name: hazelcast-cluster
      resolve-not-ready-addresses: true

5. Kubernetes 服务配置

apiVersion: v1
kind: Service
metadata:
  name: hazelcast-cluster
  namespace: production
spec:
  selector:
    app: my-app
  ports:
  - name: hazelcast
    port: 5701
  clusterIP: None

6. 高级配置：同时支持多种发现机制

private void configureHybridDiscovery(JoinConfig joinConfig) {
    HazelcastDiscoveryProperties.MulticastProperties multicast = discoveryProperties.getMulticast();
    HazelcastDiscoveryProperties.KubernetesProperties kubernetes = discoveryProperties.getKubernetes();
    
    // 同时启用两种发现机制
    if (multicast.isEnabled()) {
        joinConfig.getMulticastConfig()
            .setEnabled(true)
            .setMulticastGroup(multicast.getGroup())
            .setMulticastPort(multicast.getPort());
    }
    
    if (kubernetes.isEnabled()) {
        joinConfig.getKubernetesConfig()
            .setEnabled(true)
            .setProperty("namespace", kubernetes.getNamespace())
            .setProperty("service-name", kubernetes.getServiceName())
            .setProperty("resolve-not-ready-addresses", String.valueOf(kubernetes.isResolveNotReadyAddresses()));
    }
    
    // 如果两者都禁用，则使用默认的TCP/IP配置
    if (!multicast.isEnabled() && !kubernetes.isEnabled()) {
        joinConfig.getTcpIpConfig().setEnabled(true).addMember("localhost");
    }
}

7. 使用环境变量覆盖配置

在 Kubernetes 部署中，可以使用环境变量覆盖配置：

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: HAZELCAST_DISCOVERY_TYPE
          value: KUBERNETES
        - name: HAZELCAST_DISCOVERY_KUBERNETES_SERVICE_NAME
          value: hazelcast-cluster
        - name: HAZELCAST_DISCOVERY_KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

8. 配置验证和日志

添加配置验证和日志输出：

@Bean
public Config hazelcastConfig() {
    Config config = new Config();
    
    // ... 其他配置
    
    // 记录使用的发现机制
    String discoveryType = discoveryProperties.getType().name();
    logger.info("Using Hazelcast discovery type: {}", discoveryType);
    
    if (discoveryType.equals("KUBERNETES")) {
        KubernetesProperties k8s = discoveryProperties.getKubernetes();
        if (k8s.getNamespace() == null || k8s.getNamespace().isEmpty()) {
            logger.warn("Kubernetes namespace is not set, using default namespace");
        }
        if (k8s.getServiceName() == null || k8s.getServiceName().isEmpty()) {
            throw new IllegalStateException("Kubernetes service name must be set for Kubernetes discovery");
        }
    }
    
    return config;
}

9. 完整的属性配置参考

在 
"src/main/resources/META-INF/additional-spring-configuration-metadata.json" 中添加：

{
  "properties": [
    {
      "name": "hazelcast.discovery.type",
      "type": "com.example.config.HazelcastDiscoveryProperties$DiscoveryType",
      "description": "Hazelcast discovery mechanism type",
      "defaultValue": "KUBERNETES"
    },
    {
      "name": "hazelcast.discovery.multicast.enabled",
      "type": "java.lang.Boolean",
      "description": "Enable multicast discovery",
      "defaultValue": false
    },
    {
      "name": "hazelcast.discovery.multicast.group",
      "type": "java.lang.String",
      "description": "Multicast group address",
      "defaultValue": "224.2.2.3"
    },
    {
      "name": "hazelcast.discovery.multicast.port",
      "type": "java.lang.Integer",
      "description": "Multicast port",
      "defaultValue": 54327
    },
    {
      "name": "hazelcast.discovery.kubernetes.enabled",
      "type": "java.lang.Boolean",
      "description": "Enable Kubernetes discovery",
      "defaultValue": true
    },
    {
      "name": "hazelcast.discovery.kubernetes.namespace",
      "type": "java.lang.String",
      "description": "Kubernetes namespace",
      "defaultValue": "default"
    },
    {
      "name": "hazelcast.discovery.kubernetes.service-name",
      "type": "java.lang.String",
      "description": "Kubernetes service name for discovery",
      "defaultValue": "hazelcast-service"
    },
    {
      "name": "hazelcast.discovery.kubernetes.resolve-not-ready-addresses",
      "type": "java.lang.Boolean",
      "description": "Resolve not-ready addresses",
      "defaultValue": true
    }
  ]
}

10. 测试不同配置

测试 Kubernetes 配置

@SpringBootTest
@ActiveProfiles("prod")
public class KubernetesDiscoveryTest {

    @Autowired
    private HazelcastInstance hazelcastInstance;

    @Test
    public void testKubernetesDiscovery() {
        Cluster cluster = hazelcastInstance.getCluster();
        assertThat(cluster.getMembers()).isNotEmpty();
        
        // 验证 Kubernetes 配置
        Config config = hazelcastInstance.getConfig();
        KubernetesConfig kubernetesConfig = config.getNetworkConfig().getJoin().getKubernetesConfig();
        
        assertThat(kubernetesConfig.isEnabled()).isTrue();
        assertThat(kubernetesConfig.getProperty("service-name")).isEqualTo("hazelcast-cluster");
    }
}

测试 Multicast 配置

@SpringBootTest
@ActiveProfiles("dev")
public class MulticastDiscoveryTest {

    @Autowired
    private HazelcastInstance hazelcastInstance;

    @Test
    public void testMulticastDiscovery() {
        Cluster cluster = hazelcastInstance.getCluster();
        assertThat(cluster.getMembers()).isNotEmpty();
        
        // 验证 Multicast 配置
        Config config = hazelcastInstance.getConfig();
        MulticastConfig multicastConfig = config.getNetworkConfig().getJoin().getMulticastConfig();
        
        assertThat(multicastConfig.isEnabled()).isTrue();
        assertThat(multicastConfig.getMulticastGroup()).isEqualTo("224.2.2.3");
    }
}

11. GKE 特定配置建议

在 GKE 环境中，建议添加以下优化配置：

private void configureKubernetes(JoinConfig joinConfig) {
    HazelcastDiscoveryProperties.KubernetesProperties kubernetes = discoveryProperties.getKubernetes();
    
    joinConfig.getKubernetesConfig()
        .setEnabled(true)
        .setProperty("namespace", kubernetes.getNamespace())
        .setProperty("service-name", kubernetes.getServiceName())
        .setProperty("resolve-not-ready-addresses", String.valueOf(kubernetes.isResolveNotReadyAddresses()))
        
        // GKE 特定优化
        .setProperty("use-node-name-as-external-address", "true")
        .setProperty("service-label-name", "app")
        .setProperty("service-label-value", "hazelcast")
        .setProperty("pod-label-name", "app")
        .setProperty("pod-label-value", "hazelcast");
}

12. 安全配置

对于生产环境，添加安全配置：

@Bean
public Config hazelcastConfig() {
    Config config = new Config();
    
    // ... 发现配置
    
    // 安全配置
    config.getSecurityConfig().setEnabled(true);
    config.setLicenseKey("YOUR_LICENSE_KEY");
    
    // 启用TLS
    SSLConfig sslConfig = new SSLConfig()
        .setEnabled(true)
        .setFactoryClassName("com.hazelcast.nio.ssl.BasicSSLContextFactory")
        .setProperty("keyStore", "/path/to/keystore")
        .setProperty("keyStorePassword", "password")
        .setProperty("trustStore", "/path/to/truststore")
        .setProperty("trustStorePassword", "password");
    
    config.getNetworkConfig().setSSLConfig(sslConfig);
    
    return config;
}

总结

通过这种配置方式，您可以：

1. 灵活切换发现机制：通过配置文件在 Multicast 和 Kubernetes 之间切换
2. 环境适配：使用 Spring Profile 适配不同环境
3. 动态配置：在运行时根据环境变量调整配置
4. 混合模式：支持同时启用多种发现机制
5. GKE 优化：针对 GKE 环境提供特定优化配置
6. 安全加固：添加 TLS 和安全配置

在 GKE 环境中，推荐使用 Kubernetes 服务发现机制，因为它：

- 自动处理 Pod IP 变化
- 支持服务发现
- 与 Kubernetes 服务网格集成更好
- 提供更稳定的集群发现

对于本地开发环境，Multicast 是更简单直接的选择。这种配置方案让您可以根据环境无缝切换，无需修改代码。


package com.example.lock;

import java.util.concurrent.TimeUnit;

/**
 * 分布式锁接口
 */
public interface DistributedLock {
    
    /**
     * 获取锁（阻塞直到成功）
     */
    void lock();
    
    /**
     * 尝试获取锁（非阻塞）
     * @return true 获取成功，false 获取失败
     */
    boolean tryLock();
    
    /**
     * 尝试获取锁（带超时）
     * @param time 超时时间
     * @param unit 时间单位
     * @return true 获取成功，false 超时或失败
     * @throws InterruptedException 线程中断异常
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    
    /**
     * 释放锁
     */
    void unlock();
    
    /**
     * 检查锁是否被当前线程持有
     * @return true 当前线程持有锁，false 未持有
     */
    boolean isHeldByCurrentThread();
}



package com.example.lock.hazelcast;

import com.example.lock.DistributedLock;
import com.hazelcast.core.HazelcastInstance;
import com.hazelcast.cp.lock.ILock;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
public class HazelcastDistributedLock implements DistributedLock {

    private final HazelcastInstance hazelcastInstance;

    @Autowired
    public HazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    @Override
    public void lock() {
        getLock().lock();
    }

    @Override
    public boolean tryLock() {
        return getLock().tryLock();
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return getLock().tryLock(time, unit);
    }

    @Override
    public void unlock() {
        getLock().unlock();
    }

    @Override
    public boolean isHeldByCurrentThread() {
        return getLock().isLockedByCurrentThread();
    }

    private ILock getLock() {
        // 实际项目中应从上下文获取锁名称
        return hazelcastInstance.getCPSubsystem().getLock("default-lock");
    }
}



package com.example.lock;

/**
 * 分布式锁类型枚举
 */
public enum LockType {
    HAZELCAST,
    REDIS,
    ZOOKEEPER,
    LOCAL // 本地锁（测试用）
}


package com.example.lock;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "distributed.lock")
public class LockProperties {
    private LockType type = LockType.HAZELCAST;
    private long defaultWaitTime = 5000; // 默认等待时间(ms)
    private long defaultLeaseTime = 30000; // 默认租约时间(ms)

    // Getters and Setters
    public LockType getType() {
        return type;
    }

    public void setType(LockType type) {
        this.type = type;
    }

    public long getDefaultWaitTime() {
        return defaultWaitTime;
    }

    public void setDefaultWaitTime(long defaultWaitTime) {
        this.defaultWaitTime = defaultWaitTime;
    }

    public long getDefaultLeaseTime() {
        return defaultLeaseTime;
    }

    public void setDefaultLeaseTime(long defaultLeaseTime) {
        this.defaultLeaseTime = defaultLeaseTime;
    }
}





package com.example.lock;

import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(LockProperties.class)
public class DistributedLockAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DistributedLock distributedLock(LockProperties properties) {
        // 根据配置选择锁实现
        switch (properties.getType()) {
            case HAZELCAST:
                return hazelcastDistributedLock();
            case REDIS:
                // 未来可添加Redis实现
                throw new UnsupportedOperationException("Redis lock not implemented yet");
            case ZOOKEEPER:
                // 未来可添加Zookeeper实现
                throw new UnsupportedOperationException("Zookeeper lock not implemented yet");
            case LOCAL:
                return new LocalDistributedLock();
            default:
                throw new IllegalArgumentException("Unsupported lock type: " + properties.getType());
        }
    }

    @Bean
    @ConditionalOnClass(name = "com.hazelcast.core.HazelcastInstance")
    public HazelcastDistributedLock hazelcastDistributedLock() {
        return new HazelcastDistributedLock();
    }
    
    // 本地锁实现（用于测试）
    static class LocalDistributedLock implements DistributedLock {
        private final java.util.concurrent.locks.Lock lock = new java.util.concurrent.locks.ReentrantLock();
        
        @Override
        public void lock() {
            lock.lock();
        }

        @Override
        public boolean tryLock() {
            return lock.tryLock();
        }

        @Override
        public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
            return lock.tryLock(time, unit);
        }

        @Override
        public void unlock() {
            lock.unlock();
        }

        @Override
        public boolean isHeldByCurrentThread() {
            return ((java.util.concurrent.locks.ReentrantLock) lock).isHeldByCurrentThread();
        }
    }
}


org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.lock.DistributedLockAutoConfiguration


@Service
public class PaymentService {
    
    private final DistributedLock lock;
    
    public PaymentService(DistributedLock lock) {
        this.lock = lock;
    }
    
    public void processPayment(String accountId) {
        lock.lock();
        try {
            // 关键业务逻辑
        } finally {
            lock.unlock();
        }
    }
}


distributed:
  lock:
    type: HAZELCAST
    default-wait-time: 3000
    default-lease-time: 10000






package com.example.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;

/**
 * 分布式锁接口
 */
public interface DistributedLock extends Lock {

    /**
     * 获取锁（可重入）
     * 
     * @param key 锁的键值
     */
    void lock(String key);

    /**
     * 尝试获取锁
     * 
     * @param key 锁的键值
     * @return true 如果成功获取锁
     */
    boolean tryLock(String key);

    /**
     * 尝试获取锁（带超时）
     * 
     * @param key 锁的键值
     * @param time 超时时间
     * @param unit 时间单位
     * @return true 如果成功获取锁
     * @throws InterruptedException 如果线程被中断
     */
    boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException;

    /**
     * 释放锁
     * 
     * @param key 锁的键值
     */
    void unlock(String key);

    /**
     * 检查锁是否被持有
     * 
     * @param key 锁的键值
     * @return true 如果锁被持有
     */
    boolean isLocked(String key);
}




package com.example.lock.impl;

import com.example.lock.DistributedLock;
import com.hazelcast.core.HazelcastInstance;
import com.hazelcast.cp.lock.ILock;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
public class HazelcastDistributedLock implements DistributedLock {

    private final HazelcastInstance hazelcastInstance;
    private final ConcurrentHashMap<String, ILock> lockCache = new ConcurrentHashMap<>();

    @Autowired
    public HazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    private ILock getLock(String key) {
        return lockCache.computeIfAbsent(key, k -> 
            hazelcastInstance.getCPSubsystem().getLock(k)
        );
    }

    @Override
    public void lock(String key) {
        getLock(key).lock();
    }

    @Override
    public void lockInterruptibly(String key) throws InterruptedException {
        getLock(key).lockInterruptibly();
    }

    @Override
    public boolean tryLock(String key) {
        return getLock(key).tryLock();
    }

    @Override
    public boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException {
        return getLock(key).tryLock(time, unit);
    }

    @Override
    public void unlock(String key) {
        getLock(key).unlock();
    }

    @Override
    public Condition newCondition() {
        throw new UnsupportedOperationException("Conditions not supported in Hazelcast distributed locks");
    }

    @Override
    public boolean isLocked(String key) {
        return getLock(key).isLocked();
    }

    // 以下方法实现无key的锁操作（默认使用全局锁）
    @Override
    public void lock() {
        lock("global_lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly("global_lock");
    }

    @Override
    public boolean tryLock() {
        return tryLock("global_lock");
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock("global_lock", time, unit);
    }

    @Override
    public void unlock() {
        unlock("global_lock");
    }
}



package com.example.lock.impl;

import com.example.lock.DistributedLock;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "LOCAL", matchIfMissing = true)
public class LocalDistributedLock implements DistributedLock {

    private final ConcurrentHashMap<String, Lock> lockMap = new ConcurrentHashMap<>();

    private Lock getLock(String key) {
        return lockMap.computeIfAbsent(key, k -> new ReentrantLock());
    }

    @Override
    public void lock(String key) {
        getLock(key).lock();
    }

    @Override
    public void lockInterruptibly(String key) throws InterruptedException {
        getLock(key).lockInterruptibly();
    }

    @Override
    public boolean tryLock(String key) {
        return getLock(key).tryLock();
    }

    @Override
    public boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException {
        return getLock(key).tryLock(time, unit);
    }

    @Override
    public void unlock(String key) {
        getLock(key).unlock();
    }

    @Override
    public Condition newCondition() {
        throw new UnsupportedOperationException("Conditions not supported in local distributed lock implementation");
    }

    @Override
    public boolean isLocked(String key) {
        Lock lock = lockMap.get(key);
        return lock != null && ((ReentrantLock) lock).isLocked();
    }

    // 无key方法实现（使用全局锁）
    @Override
    public void lock() {
        lock("global_lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly("global_lock");
    }

    @Override
    public boolean tryLock() {
        return tryLock("global_lock");
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock("global_lock", time, unit);
    }

    @Override
    public void unlock() {
        unlock("global_lock");
    }
}



package com.example.lock;

import com.example.lock.impl.HazelcastDistributedLock;
import com.example.lock.impl.LocalDistributedLock;
import com.hazelcast.core.HazelcastInstance;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(LockProperties.class)
public class DistributedLockAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
    @ConditionalOnBean(HazelcastInstance.class)
    public DistributedLock hazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        return new HazelcastDistributedLock(hazelcastInstance);
    }

    @Bean
    @ConditionalOnMissingBean(DistributedLock.class)
    public DistributedLock localDistributedLock() {
        return new LocalDistributedLock();
    }
}



package com.example.lock;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "distributed.lock")
public class LockProperties {

    /**
     * 锁类型
     */
    private LockType type = LockType.LOCAL;

    /**
     * 默认等待时间（毫秒）
     */
    private long defaultWaitTime = 3000;

    /**
     * 默认租约时间（毫秒）
     */
    private long defaultLeaseTime = 10000;

    // Getters and Setters
    public LockType getType() {
        return type;
    }

    public void setType(LockType type) {
        this.type = type;
    }

    public long getDefaultWaitTime() {
        return defaultWaitTime;
    }

    public void setDefaultWaitTime(long defaultWaitTime) {
        this.defaultWaitTime = defaultWaitTime;
    }

    public long getDefaultLeaseTime() {
        return defaultLeaseTime;
    }

    public void setDefaultLeaseTime(long defaultLeaseTime) {
        this.defaultLeaseTime = defaultLeaseTime;
    }
}



package com.example.lock;

public enum LockType {
    /**
     * Hazelcast 分布式锁
     */
    HAZELCAST,
    
    /**
     * Redis 分布式锁（预留）
     */
    REDIS,
    
    /**
     * ZooKeeper 分布式锁（预留）
     */
    ZOOKEEPER,
    
    /**
     * 本地锁（用于测试或单机环境）
     */
    LOCAL
}


package com.example.lock.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    
    /**
     * 锁的key，支持SpEL表达式
     */
    String key();
    
    /**
     * 等待锁的最长时间（默认-1表示不等待）
     */
    long waitTime() default -1;
    
    /**
     * 等待时间单位（默认毫秒）
     */
    TimeUnit timeUnit() default TimeUnit.MILLISECONDS;
    
    /**
     * 锁的租约时间（默认-1表示使用配置的默认值）
     */
    long leaseTime() default -1;
    
    /**
     * 获取锁失败时的错误信息
     */
    String errorMessage() default "Failed to acquire lock";
}


package com.example.lock.aspect;

import com.example.lock.DistributedLock;
import com.example.lock.annotation.DistributedLock;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class DistributedLockAspect {

    private final DistributedLock distributedLock;
    private final SpelExpressionParser parser = new SpelExpressionParser();

    @Autowired
    public DistributedLockAspect(DistributedLock distributedLock) {
        this.distributedLock = distributedLock;
    }

    @Around("@annotation(lockAnnotation)")
    public Object around(ProceedingJoinPoint joinPoint, DistributedLock lockAnnotation) throws Throwable {
        String key = resolveKey(lockAnnotation.key(), joinPoint);
        long waitTime = lockAnnotation.waitTime();
        long leaseTime = lockAnnotation.leaseTime();
        TimeUnit timeUnit = lockAnnotation.timeUnit();
        
        boolean acquired = false;
        try {
            if (waitTime >= 0) {
                acquired = distributedLock.tryLock(key, waitTime, timeUnit);
            } else {
                distributedLock.lock(key);
                acquired = true;
            }
            
            if (!acquired) {
                throw new IllegalStateException(lockAnnotation.errorMessage());
            }
            
            return joinPoint.proceed();
        } finally {
            if (acquired) {
                distributedLock.unlock(key);
            }
        }
    }

    private String resolveKey(String keyExpression, ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Object[] args = joinPoint.getArgs();
        String[] parameterNames = signature.getParameterNames();
        
        StandardEvaluationContext context = new StandardEvaluationContext();
        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }
        
        Expression expression = parser.parseExpression(keyExpression);
        return expression.getValue(context, String.class);
    }
}




org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.lock.DistributedLockAutoConfiguration



# application.yml
distributed:
  lock:
    type: HAZELCAST
    default-wait-time: 5000
    default-lease-time: 30000

hazelcast:
  network:
    join:
      multicast:
        enabled: false
      tcp-ip:
        enabled: true
        members: [127.0.0.1]
  cp-subsystem:
    cp-member-count: 3





分布式锁 Spring Boot Starter 设计

下面是完整的分布式锁 Spring Boot Starter 实现，包含支持 key 参数的锁接口设计和 Hazelcast 实现：

文件结构

distributed-lock-spring-boot-starter/
├── src/main/java/
│   ├── com/example/lock/
│   │   ├── DistributedLock.java             # 锁接口
│   │   ├── LockProperties.java              # 配置属性
│   │   ├── LockType.java                    # 锁类型枚举
│   │   ├── DistributedLockAutoConfiguration.java # 自动配置
│   │   ├── impl/
│   │   │   ├── HazelcastDistributedLock.java # Hazelcast 实现
│   │   │   └── LocalDistributedLock.java    # 本地锁实现（用于测试或单机）
│   │   └── annotation/
│   │       └── DistributedLock.java         # 锁注解
│   └── resources/
│       └── META-INF/
│           ├── spring.factories             # 自动配置注册
│           └── additional-spring-configuration-metadata.json # 配置元数据

核心代码实现

1. 锁接口 (DistributedLock.java)

package com.example.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;

/**
 * 分布式锁接口
 */
public interface DistributedLock extends Lock {

    /**
     * 获取锁（可重入）
     * 
     * @param key 锁的键值
     */
    void lock(String key);

    /**
     * 尝试获取锁
     * 
     * @param key 锁的键值
     * @return true 如果成功获取锁
     */
    boolean tryLock(String key);

    /**
     * 尝试获取锁（带超时）
     * 
     * @param key 锁的键值
     * @param time 超时时间
     * @param unit 时间单位
     * @return true 如果成功获取锁
     * @throws InterruptedException 如果线程被中断
     */
    boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException;

    /**
     * 释放锁
     * 
     * @param key 锁的键值
     */
    void unlock(String key);

    /**
     * 检查锁是否被持有
     * 
     * @param key 锁的键值
     * @return true 如果锁被持有
     */
    boolean isLocked(String key);
}

2. Hazelcast 实现 (HazelcastDistributedLock.java)

package com.example.lock.impl;

import com.example.lock.DistributedLock;
import com.hazelcast.core.HazelcastInstance;
import com.hazelcast.cp.lock.ILock;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
public class HazelcastDistributedLock implements DistributedLock {

    private final HazelcastInstance hazelcastInstance;
    private final ConcurrentHashMap<String, ILock> lockCache = new ConcurrentHashMap<>();

    @Autowired
    public HazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    private ILock getLock(String key) {
        return lockCache.computeIfAbsent(key, k -> 
            hazelcastInstance.getCPSubsystem().getLock(k)
        );
    }

    @Override
    public void lock(String key) {
        getLock(key).lock();
    }

    @Override
    public void lockInterruptibly(String key) throws InterruptedException {
        getLock(key).lockInterruptibly();
    }

    @Override
    public boolean tryLock(String key) {
        return getLock(key).tryLock();
    }

    @Override
    public boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException {
        return getLock(key).tryLock(time, unit);
    }

    @Override
    public void unlock(String key) {
        getLock(key).unlock();
    }

    @Override
    public Condition newCondition() {
        throw new UnsupportedOperationException("Conditions not supported in Hazelcast distributed locks");
    }

    @Override
    public boolean isLocked(String key) {
        return getLock(key).isLocked();
    }

    // 以下方法实现无key的锁操作（默认使用全局锁）
    @Override
    public void lock() {
        lock("global_lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly("global_lock");
    }

    @Override
    public boolean tryLock() {
        return tryLock("global_lock");
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock("global_lock", time, unit);
    }

    @Override
    public void unlock() {
        unlock("global_lock");
    }
}

3. 本地锁实现 (LocalDistributedLock.java)

package com.example.lock.impl;

import com.example.lock.DistributedLock;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "LOCAL", matchIfMissing = true)
public class LocalDistributedLock implements DistributedLock {

    private final ConcurrentHashMap<String, Lock> lockMap = new ConcurrentHashMap<>();

    private Lock getLock(String key) {
        return lockMap.computeIfAbsent(key, k -> new ReentrantLock());
    }

    @Override
    public void lock(String key) {
        getLock(key).lock();
    }

    @Override
    public void lockInterruptibly(String key) throws InterruptedException {
        getLock(key).lockInterruptibly();
    }

    @Override
    public boolean tryLock(String key) {
        return getLock(key).tryLock();
    }

    @Override
    public boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException {
        return getLock(key).tryLock(time, unit);
    }

    @Override
    public void unlock(String key) {
        getLock(key).unlock();
    }

    @Override
    public Condition newCondition() {
        throw new UnsupportedOperationException("Conditions not supported in local distributed lock implementation");
    }

    @Override
    public boolean isLocked(String key) {
        Lock lock = lockMap.get(key);
        return lock != null && ((ReentrantLock) lock).isLocked();
    }

    // 无key方法实现（使用全局锁）
    @Override
    public void lock() {
        lock("global_lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly("global_lock");
    }

    @Override
    public boolean tryLock() {
        return tryLock("global_lock");
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock("global_lock", time, unit);
    }

    @Override
    public void unlock() {
        unlock("global_lock");
    }
}

4. 自动配置 (DistributedLockAutoConfiguration.java)

package com.example.lock;

import com.example.lock.impl.HazelcastDistributedLock;
import com.example.lock.impl.LocalDistributedLock;
import com.hazelcast.core.HazelcastInstance;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(LockProperties.class)
public class DistributedLockAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
    @ConditionalOnBean(HazelcastInstance.class)
    public DistributedLock hazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        return new HazelcastDistributedLock(hazelcastInstance);
    }

    @Bean
    @ConditionalOnMissingBean(DistributedLock.class)
    public DistributedLock localDistributedLock() {
        return new LocalDistributedLock();
    }
}

5. 配置属性 (LockProperties.java)

package com.example.lock;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "distributed.lock")
public class LockProperties {

    /**
     * 锁类型
     */
    private LockType type = LockType.LOCAL;

    /**
     * 默认等待时间（毫秒）
     */
    private long defaultWaitTime = 3000;

    /**
     * 默认租约时间（毫秒）
     */
    private long defaultLeaseTime = 10000;

    // Getters and Setters
    public LockType getType() {
        return type;
    }

    public void setType(LockType type) {
        this.type = type;
    }

    public long getDefaultWaitTime() {
        return defaultWaitTime;
    }

    public void setDefaultWaitTime(long defaultWaitTime) {
        this.defaultWaitTime = defaultWaitTime;
    }

    public long getDefaultLeaseTime() {
        return defaultLeaseTime;
    }

    public void setDefaultLeaseTime(long defaultLeaseTime) {
        this.defaultLeaseTime = defaultLeaseTime;
    }
}

6. 锁类型枚举 (LockType.java)

package com.example.lock;

public enum LockType {
    /**
     * Hazelcast 分布式锁
     */
    HAZELCAST,
    
    /**
     * Redis 分布式锁（预留）
     */
    REDIS,
    
    /**
     * ZooKeeper 分布式锁（预留）
     */
    ZOOKEEPER,
    
    /**
     * 本地锁（用于测试或单机环境）
     */
    LOCAL
}

7. 锁注解 (DistributedLock.java)

package com.example.lock.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    
    /**
     * 锁的key，支持SpEL表达式
     */
    String key();
    
    /**
     * 等待锁的最长时间（默认-1表示不等待）
     */
    long waitTime() default -1;
    
    /**
     * 等待时间单位（默认毫秒）
     */
    TimeUnit timeUnit() default TimeUnit.MILLISECONDS;
    
    /**
     * 锁的租约时间（默认-1表示使用配置的默认值）
     */
    long leaseTime() default -1;
    
    /**
     * 获取锁失败时的错误信息
     */
    String errorMessage() default "Failed to acquire lock";
}

8. 切面实现（可选）

package com.example.lock.aspect;

import com.example.lock.DistributedLock;
import com.example.lock.annotation.DistributedLock;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class DistributedLockAspect {

    private final DistributedLock distributedLock;
    private final SpelExpressionParser parser = new SpelExpressionParser();

    @Autowired
    public DistributedLockAspect(DistributedLock distributedLock) {
        this.distributedLock = distributedLock;
    }

    @Around("@annotation(lockAnnotation)")
    public Object around(ProceedingJoinPoint joinPoint, DistributedLock lockAnnotation) throws Throwable {
        String key = resolveKey(lockAnnotation.key(), joinPoint);
        long waitTime = lockAnnotation.waitTime();
        long leaseTime = lockAnnotation.leaseTime();
        TimeUnit timeUnit = lockAnnotation.timeUnit();
        
        boolean acquired = false;
        try {
            if (waitTime >= 0) {
                acquired = distributedLock.tryLock(key, waitTime, timeUnit);
            } else {
                distributedLock.lock(key);
                acquired = true;
            }
            
            if (!acquired) {
                throw new IllegalStateException(lockAnnotation.errorMessage());
            }
            
            return joinPoint.proceed();
        } finally {
            if (acquired) {
                distributedLock.unlock(key);
            }
        }
    }

    private String resolveKey(String keyExpression, ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Object[] args = joinPoint.getArgs();
        String[] parameterNames = signature.getParameterNames();
        
        StandardEvaluationContext context = new StandardEvaluationContext();
        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }
        
        Expression expression = parser.parseExpression(keyExpression);
        return expression.getValue(context, String.class);
    }
}

9. 自动配置注册 (spring.factories)

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.lock.DistributedLockAutoConfiguration

10. 配置元数据 (additional-spring-configuration-metadata.json)

{
  "properties": [
    {
      "name": "distributed.lock.type",
      "type": "com.example.lock.LockType",
      "description": "分布式锁实现类型",
      "defaultValue": "LOCAL"
    },
    {
      "name": "distributed.lock.default-wait-time",
      "type": "java.lang.Long",
      "description": "默认等待锁的时间(毫秒)",
      "defaultValue": 3000
    },
    {
      "name": "distributed.lock.default-lease-time",
      "type": "java.lang.Long",
      "description": "默认锁租约时间(毫秒)",
      "defaultValue": 10000
    }
  ]
}

使用示例

1. 添加依赖

<dependency>
    <groupId>com.example</groupId>
    <artifactId>distributed-lock-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>

2. 配置 Hazelcast

# application.yml
distributed:
  lock:
    type: HAZELCAST
    default-wait-time: 5000
    default-lease-time: 30000

hazelcast:
  network:
    join:
      multicast:
        enabled: false
      tcp-ip:
        enabled: true
        members: [127.0.0.1]
  cp-subsystem:
    cp-member-count: 3

3. 在代码中使用

方式一：直接注入 DistributedLock

@Service
public class OrderService {

    private final DistributedLock lock;

    public OrderService(DistributedLock lock) {
        this.lock = lock;
    }

    public void processOrder(String orderId) {
        String lockKey = "order_lock_" + orderId;
        
        try {
            if (lock.tryLock(lockKey, 3, TimeUnit.SECONDS)) {
                try {
                    // 处理订单业务
                } finally {
                    lock.unlock(lockKey);
                }
            } else {
                throw new RuntimeException("Failed to acquire lock for order: " + orderId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock acquisition interrupted", e);
        }
    }
}

方式二：使用注解

@Service
public class PaymentService {

    @DistributedLock(
        key = "'payment_lock_' + #accountId", 
        waitTime = 5000,
        errorMessage = "Payment processing is busy, please try again later"
    )
    public void processPayment(String accountId, BigDecimal amount) {
        // 支付处理逻辑
    }
}

设计特点

1. 接口驱动设计：通过 
"DistributedLock" 接口抽象锁操作，支持多种实现
2. Key 参数支持：所有锁操作都支持传入 key，实现细粒度锁控制
3. 条件装配：根据配置动态选择锁实现
4. 本地锁支持：提供 
"LocalDistributedLock" 用于测试环境
5. 注解支持：通过 AOP 实现声明式锁
6. 配置驱动：支持通过配置文件调整锁参数
7. SPEL 表达式：注解支持 SpEL 表达式动态生成 key
8. 自动配置：Spring Boot Starter 自动装配机制
9. 缓存优化：缓存锁对象减少 Hazelcast 网络开销
10. 无侵入设计：业务代码无需关注具体锁实现

这个设计提供了灵活、可扩展的分布式锁解决方案，可以轻松集成到 Spring Boot 项目中，并支持未来扩展其他分布式锁实现。








#——————————————————————————————————————————————

您说得对，在 Hazelcast 5.5.0 中，
"ILock" 类已被弃用并替换为 
"FencedLock"。感谢您的指正！以下是更新后的 Hazelcast 分布式锁实现，使用 Hazelcast 5.5.0 的 
"FencedLock"：

package com.example.lock.impl;

import com.example.lock.DistributedLock;
import com.hazelcast.core.HazelcastInstance;
import com.hazelcast.cp.lock.FencedLock;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;

@Component
@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "HAZELCAST")
public class HazelcastDistributedLock implements DistributedLock {

    private final HazelcastInstance hazelcastInstance;
    private final ConcurrentHashMap<String, FencedLock> lockCache = new ConcurrentHashMap<>();

    @Autowired
    public HazelcastDistributedLock(HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    private FencedLock getLock(String key) {
        return lockCache.computeIfAbsent(key, k -> 
            hazelcastInstance.getCPSubsystem().getLock(k)
        );
    }

    @Override
    public void lock(String key) {
        getLock(key).lock();
    }

    @Override
    public void lockInterruptibly(String key) throws InterruptedException {
        getLock(key).lockInterruptibly();
    }

    @Override
    public boolean tryLock(String key) {
        return getLock(key).tryLock();
    }

    @Override
    public boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException {
        return getLock(key).tryLock(time, unit);
    }

    @Override
    public void unlock(String key) {
        getLock(key).unlock();
    }

    @Override
    public Condition newCondition() {
        throw new UnsupportedOperationException("Conditions not supported in Hazelcast distributed locks");
    }

    @Override
    public boolean isLocked(String key) {
        return getLock(key).isLocked();
    }

    @Override
    public long getLockCount(String key) {
        return getLock(key).getLockCount();
    }

    @Override
    public boolean isLockedByCurrentThread(String key) {
        return getLock(key).isLockedByCurrentThread();
    }

    // 以下方法实现无key的锁操作（默认使用全局锁）
    @Override
    public void lock() {
        lock("global_lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly("global_lock");
    }

    @Override
    public boolean tryLock() {
        return tryLock("global_lock");
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock("global_lock", time, unit);
    }

    @Override
    public void unlock() {
        unlock("global_lock");
    }
}

更新后的分布式锁接口

为了更好支持 Hazelcast 5.5.0 的特性，我扩展了锁接口：

package com.example.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;

/**
 * 分布式锁接口
 */
public interface DistributedLock extends Lock {

    /**
     * 获取锁（可重入）
     * 
     * @param key 锁的键值
     */
    void lock(String key);

    /**
     * 尝试获取锁
     * 
     * @param key 锁的键值
     * @return true 如果成功获取锁
     */
    boolean tryLock(String key);

    /**
     * 尝试获取锁（带超时）
     * 
     * @param key 锁的键值
     * @param time 超时时间
     * @param unit 时间单位
     * @return true 如果成功获取锁
     * @throws InterruptedException 如果线程被中断
     */
    boolean tryLock(String key, long time, TimeUnit unit) throws InterruptedException;

    /**
     * 释放锁
     * 
     * @param key 锁的键值
     */
    void unlock(String key);

    /**
     * 检查锁是否被持有
     * 
     * @param key 锁的键值
     * @return true 如果锁被持有
     */
    boolean isLocked(String key);
    
    /**
     * 获取当前线程持有锁的次数
     * 
     * @param key 锁的键值
     * @return 持有锁的次数
     */
    long getLockCount(String key);
    
    /**
     * 检查锁是否被当前线程持有
     * 
     * @param key 锁的键值
     * @return true 如果锁被当前线程持有
     */
    boolean isLockedByCurrentThread(String key);
}

Hazelcast 5.5.0 配置注意事项

在 Hazelcast 5.5.0 中，CP 子系统的配置有所变化：

hazelcast:
  cp-subsystem:
    cp-member-count: 3 # Raft 组节点数（建议奇数）
    group-size: 3      # 与 cp-member-count 相同
    session-time-to-live-seconds: 300
    missing-cp-member-auto-removal-seconds: 14400
    fail-on-indeterminate-operation-state: false
    data-load-timeout-seconds: 300

使用 FencedLock 的高级特性

Hazelcast 的 
"FencedLock" 提供了额外的功能：

// 获取锁的fence token（用于顺序保证）
long fenceToken = lock.getFence();

// 检查当前锁的fence token
long currentFence = lock.getFence();

// 使用锁保护临界区
lock.lock();
try {
    // 业务逻辑
} finally {
    lock.unlock();
}

// 尝试锁并获取fence token
Long fence = lock.tryLockAndGetFence();
if (fence != null) {
    try {
        // 业务逻辑
    } finally {
        lock.unlock();
    }
}

自动配置更新

确保 Hazelcast 实例已配置 CP 子系统：

@Configuration
public class HazelcastConfig {

    @Bean
    public Config hazelcastConfig(LockProperties lockProperties) {
        Config config = new Config();
        
        // 配置 CP 子系统
        CPSubsystemConfig cpSubsystemConfig = new CPSubsystemConfig();
        cpSubsystemConfig.setCPMemberCount(lockProperties.getCpMemberCount());
        cpSubsystemConfig.setGroupSize(lockProperties.getCpMemberCount());
        cpSubsystemConfig.setSessionTimeToLiveSeconds(lockProperties.getSessionTTL());
        config.setCPSubsystemConfig(cpSubsystemConfig);
        
        // 其他配置...
        return config;
    }
}

锁工具类（使用 FencedLock）

@Component
public class LockUtil {

    private final DistributedLock lock;

    public LockUtil(DistributedLock lock) {
        this.lock = lock;
    }

    public <T> T executeWithLock(String key, Supplier<T> supplier) {
        lock.lock(key);
        try {
            return supplier.get();
        } finally {
            lock.unlock(key);
        }
    }

    public void executeWithLock(String key, Runnable runnable) {
        lock.lock(key);
        try {
            runnable.run();
        } finally {
            lock.unlock(key);
        }
    }

    public <T> T tryExecuteWithLock(String key, long timeout, TimeUnit unit, Supplier<T> supplier) {
        try {
            if (lock.tryLock(key, timeout, unit)) {
                try {
                    return supplier.get();
                } finally {
                    lock.unlock(key);
                }
            }
            return null;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
}

最佳实践

1. 锁粒度：使用业务特定键值（如用户ID、订单ID）而非全局锁
2. 超时设置：总是使用带超时的 
"tryLock" 方法
3. 锁释放：在 finally 块中确保锁释放
4. 重入检查：使用 
"isLockedByCurrentThread" 避免重复加锁
5. CP 配置：确保 CP 子系统正确配置（奇数节点）
6. 监控：监控锁的持有时间和争用情况

这个更新后的实现完全兼容 Hazelcast 5.5.0，并利用了新的 
"FencedLock" API 提供的功能。





import com.hazelcast.core.HazelcastInstance;
import com.hazelcast.map.IMap;
import org.springframework.stereotype.Component;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Component
public class HazelcastAPLock implements APDistributedLock {

    private final HazelcastInstance hazelcastInstance;
    private final ThreadLocal<String> threadLockHolder = new ThreadLocal<>();
    private static final String LOCK_MAP_NAME = "distributed_locks";

    public HazelcastAPLock(HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    private IMap<String, String> getLockMap() {
        return hazelcastInstance.getMap(LOCK_MAP_NAME);
    }

    @Override
    public boolean tryLock(String lockKey, long leaseTime) {
        String lockId = UUID.randomUUID().toString();
        boolean acquired = getLockMap().tryLock(lockKey, leaseTime, TimeUnit.MILLISECONDS);
        if (acquired) {
            threadLockHolder.set(lockId);
            getLockMap().put(lockKey, lockId);
        }
        return acquired;
    }

    @Override
    public void unlock(String lockKey) {
        String currentLockId = threadLockHolder.get();
        String lockIdInMap = getLockMap().get(lockKey);
        
        if (currentLockId != null && currentLockId.equals(lockIdInMap)) {
            getLockMap().unlock(lockKey);
            threadLockHolder.remove();
        }
    }

    @Override
    public boolean tryLock(String lockKey, long leaseTime, long waitTime) throws InterruptedException {
        String lockId = UUID.randomUUID().toString();
        boolean acquired = getLockMap().tryLock(lockKey, waitTime, TimeUnit.MILLISECONDS, leaseTime, TimeUnit.MILLISECONDS);
        if (acquired) {
            threadLockHolder.set(lockId);
            getLockMap().put(lockKey, lockId);
        }
        return acquired;
    }

    @Override
    public boolean isLocked(String lockKey) {
        return getLockMap().isLocked(lockKey);
    }
}


