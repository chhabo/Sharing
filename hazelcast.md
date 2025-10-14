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


