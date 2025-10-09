3实现步骤
1. 添加延迟测试服务类

```java
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.*;

@Service
public class NetworkLatencyService {

    private static final int DEFAULT_TIMEOUT = 5000; // 5秒超时

    public LatencyReport testNetworkLatency(String host, int port) {
        int testCount = 10;
        int threadCount = 5;
        int stabilityDuration = 30; // 秒

        List<Long> sequentialResults = runSequentialTests(host, port, testCount);
        List<Long> concurrentResults = runConcurrentTests(host, port, testCount, threadCount);
        StabilityResult stabilityResult = testConnectionStability(host, port, stabilityDuration);

        return new LatencyReport(
                calculateStats(sequentialResults),
                calculateStats(concurrentResults),
                stabilityResult
        );
    }

    public List<Long> runSequentialTests(String host, int port, int count) {
        List<Long> results = new ArrayList<>();

        for (int i = 0; i < count; i++) {
            try {
                long latency = measureConnectionTime(host, port);
                results.add(latency);
            } catch (IOException e) {
                // 记录错误但继续测试
            }

            try {
                Thread.sleep(200);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }

        return results;
    }

    public List<Long> runConcurrentTests(String host, int port, int count, int threadCount) {
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        List<Future<Long>> futures = new ArrayList<>();
        List<Long> results = new ArrayList<>();

        for (int i = 0; i < count; i++) {
            futures.add(executor.submit(() -> measureConnectionTime(host, port)));
        }

        for (Future<Long> future : futures) {
            try {
                Long latency = future.get(10, TimeUnit.SECONDS);
                results.add(latency);
            } catch (Exception e) {
                // 记录错误
            }
        }

        executor.shutdown();
        return results;
    }

    public StabilityResult testConnectionStability(String host, int port, int durationSeconds) {
        try (Socket socket = new Socket()) {
            long startTime = System.currentTimeMillis();
            socket.connect(new InetSocketAddress(host, port), DEFAULT_TIMEOUT);
            long connectTime = System.currentTimeMillis() - startTime;

            // 保持连接指定时间
            for (int i = 0; i < durationSeconds; i++) {
                Thread.sleep(1000);
                if (!socket.isConnected() || socket.isClosed()) {
                    return new StabilityResult(connectTime, false, "Connection dropped at " + i + " seconds");
                }
            }

            return new StabilityResult(connectTime, true, null);
        } catch (Exception e) {
            return new StabilityResult(0, false, "Stability test failed: " + e.getMessage());
        }
    }

    private long measureConnectionTime(String host, int port) throws IOException {
        try (Socket socket = new Socket()) {
            long startTime = System.currentTimeMillis();
            socket.connect(new InetSocketAddress(host, port), DEFAULT_TIMEOUT);
            return System.currentTimeMillis() - startTime;
        }
    }

    private Stats calculateStats(List<Long> results) {
        if (results == null || results.isEmpty()) {
            return new Stats(Collections.emptyList());
        }

        Collections.sort(results);
        long min = results.get(0);
        long max = results.get(results.size() - 1);
        double avg = results.stream().mapToLong(l -> l).average().orElse(0);

        double median;
        int size = results.size();
        if (size % 2 == 0) {
            median = (results.get(size/2 - 1) + results.get(size/2)) / 2.0;
        } else {
            median = results.get(size/2);
        }

        return new Stats(results, min, max, avg, median);
    }

    // 内部数据类
    public record Stats(List<Long> results, long min, long max, double avg, double median) {
        public Stats(List<Long> results) {
            this(results,
                    results.isEmpty() ? 0 : Collections.min(results),
                    results.isEmpty() ? 0 : Collections.max(results),
                    results.isEmpty() ? 0 : results.stream().mapToLong(l -> l).average().orElse(0),
                    results.isEmpty() ? 0 : calculateMedian(results));
        }

        private static double calculateMedian(List<Long> values) {
            Collections.sort(values);
            int size = values.size();
            if (size % 2 == 0) {
                return (values.get(size/2 - 1) + values.get(size/2)) / 2.0;
            } else {
                return values.get(size/2);
            }
        }
    }

    public record StabilityResult(long initialLatency, boolean success, String error) {}

    public record LatencyReport(Stats sequentialStats, Stats concurrentStats, StabilityResult stabilityResult) {
        public String getReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== Network Latency Test Report ===\n");

            sb.append("\nSequential Connection Test:\n");
            appendStats(sb, sequentialStats);

            sb.append("\nConcurrent Connection Test:\n");
            appendStats(sb, concurrentStats);

            sb.append("\nConnection Stability Test:\n");
            sb.append("  Initial connection: ").append(stabilityResult.initialLatency()).append(" ms\n");
            if (stabilityResult.success()) {
                sb.append("  Connection maintained successfully\n");
            } else {
                sb.append("  Connection failed: ").append(stabilityResult.error()).append("\n");
            }

            return sb.toString();
        }

        private void appendStats(StringBuilder sb, Stats stats) {
            sb.append("  Tests performed: ").append(stats.results().size()).append("\n");
            sb.append("  Minimum latency: ").append(stats.min()).append(" ms\n");
            sb.append("  Maximum latency: ").append(stats.max()).append(" ms\n");
            sb.append("  Average latency: ").append(String.format("%.2f", stats.avg())).append(" ms\n");
            sb.append("  Median latency:  ").append(String.format("%.2f", stats.median())).append(" ms\n");
        }
    }
}
```


```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NetworkLatencyController {

    private final NetworkLatencyService latencyService;
    
    // 从配置中获取默认数据库地址
    @Value("${spring.datasource.url}")
    private String datasourceUrl;
    
    @Autowired
    public NetworkLatencyController(NetworkLatencyService latencyService) {
        this.latencyService = latencyService;
    }
    
    @GetMapping("/api/network/latency")
    public String testNetworkLatency(
            @RequestParam(value = "host", required = false) String host,
            @RequestParam(value = "port", defaultValue = "1521") int port) {
        
        // 如果没有提供host，尝试从数据源URL中提取
        if (host == null || host.isEmpty()) {
            host = extractHostFromDatasourceUrl();
        }
        
        if (host == null) {
            return "Error: Database host not specified and could not be determined from configuration";
        }
        
        NetworkLatencyService.LatencyReport report = latencyService.testNetworkLatency(host, port);
        return report.getReport();
    }
    
    private String extractHostFromDatasourceUrl() {
        // 简单解析JDBC URL格式：jdbc:oracle:thin:@host:port:SID
        if (datasourceUrl != null && datasourceUrl.startsWith("jdbc:oracle:thin:@")) {
            String[] parts = datasourceUrl.split("@|:");
            if (parts.length >= 2) {
                return parts[1];
            }
        }
        return null;
    }
}
```



@Configuration
public class DruidConfig {

    @Bean
    public ServletRegistrationBean<StatViewServlet> druidStatViewServlet() {
        ServletRegistrationBean<StatViewServlet> reg = new ServletRegistrationBean<>(
            new StatViewServlet(), "/druid/*"
        );
        
        // 添加监控页面访问控制
        reg.addInitParameter("loginUsername", "admin"); // 监控页登录用户名
        reg.addInitParameter("loginPassword", "druid123"); // 监控页登录密码
        reg.addInitParameter("resetEnable", "false"); // 禁用重置功能
        
        // IP白名单（没有配置则允许所有访问）
        reg.addInitParameter("allow", "127.0.0.1,192.168.1.100");
        
        // IP黑名单（优先级高于白名单）
        // reg.addInitParameter("deny", "192.168.1.73");
        
        return reg;
    }

    @Bean
    public FilterRegistrationBean<WebStatFilter> druidWebStatFilter() {
        FilterRegistrationBean<WebStatFilter> reg = new FilterRegistrationBean<>(
            new WebStatFilter()
        );
        
        // 添加URL过滤规则
        reg.addUrlPatterns("/*");
        
        // 排除静态资源和不需要监控的请求
        reg.addInitParameter("exclusions", 
            "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        
        // 开启session统计功能
        reg.addInitParameter("sessionStatEnable", "true");
        
        // 配置profileEnable能够监控单个URL调用的SQL列表
        reg.addInitParameter("profileEnable", "true");
        
        return reg;
    }
}

@Bean
@ConfigurationProperties("spring.datasource.druid")
public DataSource druidDataSource() {
    DruidDataSource ds = new DruidDataSource();
    
    // 开启监控统计功能
    ds.setFilters("stat,wall");
    
    // SQL防火墙配置
    WallConfig wallConfig = new WallConfig();
    wallConfig.setDropTableAllow(false); // 禁止删表
    WallFilter wallFilter = new WallFilter();
    wallFilter.setConfig(wallConfig);
    
    // 添加过滤器
    try {
        ds.getProxyFilters().add(wallFilter);
    } catch (SQLException e) {
        throw new RuntimeException("Druid filter init failed");
    }
    
    return ds;
}

import com.alibaba.druid.spring.boot3.autoconfigure.DruidDataSourceAutoConfigure;
import com.alibaba.druid.spring.boot3.autoconfigure.properties.DruidStatProperties;
import com.alibaba.druid.support.http.StatViewServlet;
import com.alibaba.druid.support.http.WebStatFilter;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@Configuration
@AutoConfigureAfter(DruidDataSourceAutoConfigure.class) // 确保在数据源初始化后配置
public class DruidMonitorConfig {

    /**
     * 配置监控页面 Servlet
     */
    @Bean
    public ServletRegistrationBean<StatViewServlet> druidStatViewServlet() {
        ServletRegistrationBean<StatViewServlet> reg = new ServletRegistrationBean<>(
                new StatViewServlet(), "/druid/*");
        
        // 安全配置
        Map<String, String> initParams = new HashMap<>();
        initParams.put("loginUsername", "druidAdmin"); // 监控页登录账号
        initParams.put("loginPassword", "securePass123!"); // 监控页登录密码
        initParams.put("resetEnable", "false"); // 禁用重置功能
        
        // 访问控制（生产环境必须配置）
        initParams.put("allow", "127.0.0.1,192.168.1.0/24"); 
        initParams.put("deny", "192.168.1.100"); 
        
        reg.setInitParameters(initParams);
        return reg;
    }

    /**
     * 配置 Web 监控过滤器
     */
    @Bean
    public FilterRegistrationBean<WebStatFilter> druidWebStatFilter() {
        FilterRegistrationBean<WebStatFilter> reg = new FilterRegistrationBean<>();
        reg.setFilter(new WebStatFilter());
        
        Map<String, String> initParams = new HashMap<>();
        initParams.put("exclusions", "*.js,*.css,*.ico,/druid/*"); // 排除静态资源
        initParams.put("sessionStatEnable", "true"); // 开启 session 统计
        initParams.put("profileEnable", "true"); // 开启单个 URL 的 SQL 监控
        
        reg.setInitParameters(initParams);
        reg.setUrlPatterns(Collections.singletonList("/*"));
        return reg;
    }

    /**
     * 配置 SQL 防火墙（可选但推荐）
     */
    @Bean
    public DruidStatProperties druidStatProperties() {
        DruidStatProperties properties = new DruidStatProperties();
        properties.setFilter("stat,wall,slf4j"); // 启用统计、防火墙和日志
        return properties;
    }
}



<dependencies>
    <!-- Hazelcast 核心依赖 -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
        <version>5.3.6</version>
    </dependency>
    
    <!-- Spring Boot 集成支持 -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-spring</artifactId>
        <version>5.3.6</version>
    </dependency>
    
    <!-- Spring Boot 缓存支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
</dependencies>


import com.hazelcast.config.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HazelcastConfig {

    @Bean
    public Config hazelcastConfiguration() {
        Config config = new Config();
        config.setClusterName("dev-cluster");
        
        // 网络配置 - 使用默认组播发现
        NetworkConfig network = config.getNetworkConfig();
        network.setPort(5701).setPortAutoIncrement(true);
        
        JoinConfig join = network.getJoin();
        join.getMulticastConfig().setEnabled(true); // 启用组播发现
        
        // 缓存配置示例
        MapConfig mapConfig = new MapConfig();
        mapConfig.setName("defaultCache");
        mapConfig.setTimeToLiveSeconds(300); // 5分钟过期
        config.addMapConfig(mapConfig);
        
        return config;
    }
}

# 使用 Hazelcast 作为缓存提供者
spring.cache.type=hazelcast

# 配置 Hazelcast 实例名称（可选）
spring.hazelcast.instance-name=my-hazelcast-instance

# 缓存管理器配置
spring.cache.hazelcast.config=classpath:hazelcast.xml


management.endpoints.web.exposure.include=health,info,hazelcast


@Bean
public Config hazelcastConfiguration() {
    Config config = new Config();
    
    // 用户缓存配置
    MapConfig userCache = new MapConfig();
    userCache.setName("userCache");
    userCache.setTimeToLiveSeconds(1800); // 30分钟
    userCache.setMaxSizeConfig(new MaxSizeConfig(1000, MaxSizePolicy.PER_NODE));
    
    // 产品缓存配置
    MapConfig productCache = new MapConfig();
    productCache.setName("productCache");
    productCache.setTimeToLiveSeconds(3600); // 1小时
    productCache.setEvictionConfig(new EvictionConfig()
        .setEvictionPolicy(EvictionPolicy.LRU)
        .setMaxSizePolicy(MaxSizePolicy.PER_NODE)
        .setSize(2000));
    
    config.addMapConfig(userCache);
    config.addMapConfig(productCache);
    
    return config;
}














Hazelcast 在 GKE 上嵌入 Spring Boot 应用的完整指南

1. 整体架构图

graph TD
    subgraph GKE Cluster
        subgraph Pod 1
            A[Spring Boot App] --> B[Hazelcast Instance]
        end
        subgraph Pod 2
            C[Spring Boot App] --> D[Hazelcast Instance]
        end
        subgraph Pod 3
            E[Spring Boot App] --> F[Hazelcast Instance]
        end
    end
    
    B -->|TCP 5701| D
    D -->|TCP 5701| F
    F -->|TCP 5701| B
    
    G[Hazelcast Headless Service] -->|DNS Discovery| Pod1
    G -->|DNS Discovery| Pod2
    G -->|DNS Discovery| Pod3

2. 端口配置要求

2.1 必须开放的端口

端口 协议 方向 用途
5701 TCP 入站/出站 Hazelcast 节点间通信（默认端口）
5701 TCP 入站 Hazelcast 管理控制台（可选）
8080 TCP 入站 Spring Boot 应用端口（HTTP）
8443 TCP 入站 Spring Boot 应用端口（HTTPS）

2.2 可选端口

端口 协议 方向 用途
8081 TCP 入站 Hazelcast 管理控制台（默认）
9000 TCP 入站 Prometheus 监控端点
8778 TCP 入站 JMX 监控

3. Spring Boot 集成步骤

3.1 添加依赖

<dependencies>
    <!-- Hazelcast 核心 -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
        <version>5.3.6</version>
    </dependency>
    
    <!-- Spring Boot 集成 -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-spring</artifactId>
        <version>5.3.6</version>
    </dependency>
    
    <!-- Kubernetes 发现 -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-kubernetes</artifactId>
        <version>5.3.6</version>
    </dependency>
    
    <!-- Spring Boot 缓存支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
</dependencies>

3.2 Hazelcast 配置类

import com.hazelcast.config.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HazelcastConfig {

    @Value("${spring.application.name}")
    private String appName;

    @Bean
    public Config hazelcastConfiguration() {
        Config config = new Config();
        config.setClusterName(appName + "-cluster");
        
        // 网络配置
        NetworkConfig networkConfig = config.getNetworkConfig();
        networkConfig.setPort(5701)
                   .setPortAutoIncrement(true);
        
        // 启用 Kubernetes 发现
        JoinConfig joinConfig = networkConfig.getJoin();
        joinConfig.getMulticastConfig().setEnabled(false);
        joinConfig.getKubernetesConfig()
                 .setEnabled(true)
                 .setProperty("service-dns", "${service.name}.${namespace}.svc.cluster.local");
        
        // 缓存配置示例
        MapConfig mapConfig = new MapConfig();
        mapConfig.setName("defaultCache");
        mapConfig.setTimeToLiveSeconds(300);
        config.addMapConfig(mapConfig);
        
        return config;
    }
}

3.3 应用配置 (application.yaml)

spring:
  application:
    name: my-app
  cache:
    type: hazelcast

hazelcast:
  kubernetes:
    namespace: ${KUBERNETES_NAMESPACE:default}
    service-name: hazelcast-service

4. Kubernetes 部署配置

4.1 Deployment 配置

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-springboot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-springboot-app
  template:
    metadata:
      labels:
        app: my-springboot-app
    spec:
      serviceAccountName: hazelcast-sa  # 需要RBAC权限
      containers:
      - name: app
        image: gcr.io/your-project/my-springboot-app:latest
        ports:
        - containerPort: 8080  # Spring Boot HTTP端口
        - containerPort: 5701  # Hazelcast 通信端口
        env:
        - name: HAZELCAST_KUBERNETES_SERVICE_NAME
          value: "hazelcast-service"
        - name: HAZELCAST_KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: JAVA_OPTS
          value: "-Dhazelcast.kubernetes.service-dns=hazelcast-service.$(HAZELCAST_KUBERNETES_NAMESPACE).svc.cluster.local"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

4.2 Headless Service 配置

apiVersion: v1
kind: Service
metadata:
  name: hazelcast-service
spec:
  clusterIP: None  # Headless服务
  selector:
    app: my-springboot-app
  ports:
  - name: hazelcast
    port: 5701
    targetPort: 5701

4.3 RBAC 配置

apiVersion: v1
kind: ServiceAccount
metadata:
  name: hazelcast-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: hazelcast-role
rules:
- apiGroups: [""]
  resources: ["pods", "endpoints"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hazelcast-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: hazelcast-role
subjects:
- kind: ServiceAccount
  name: hazelcast-sa

5. 网络策略配置

5.1 网络策略 (NetworkPolicy)

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hazelcast-network-policy
spec:
  podSelector:
    matchLabels:
      app: my-springboot-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: my-springboot-app
    ports:
    - protocol: TCP
      port: 5701
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: my-springboot-app
    ports:
    - protocol: TCP
      port: 5701

5.2 GKE 防火墙规则

# 创建防火墙规则允许Hazelcast通信
gcloud compute firewall-rules create hazelcast-internal \
    --allow tcp:5701 \
    --source-tags gke-my-cluster \
    --target-tags gke-my-cluster \
    --network default

6. 验证集群形成

6.1 日志检查

kubectl logs <pod-name> | grep Hazelcast

预期输出：

Members {size:3, ver:3} [
    Member [10.0.0.1]:5701 - 12345678-1234-1234-1234-1234567890ab
    Member [10.0.0.2]:5701 - 23456789-2345-2345-2345-234567890bc
    Member [10.0.0.3]:5701 - 34567890-3456-3456-3456-34567890cde
]

6.2 管理控制台集成

@Bean
public Config hazelcastConfiguration() {
    Config config = new Config();
    // ... 其他配置
    
    // 启用管理中心
    ManagementCenterConfig managementCenterConfig = new ManagementCenterConfig();
    managementCenterConfig.setEnabled(true)
            .setUrl("http://hazelcast-mancenter:8080/hazelcast-mancenter");
    config.setManagementCenterConfig(managementCenterConfig);
    
    return config;
}

7. 高级配置选项

7.1 多区域部署

@Bean
public Config hazelcastConfiguration() {
    Config config = new Config();
    
    // 启用多区域部署
    config.getNetworkConfig().getJoin().getKubernetesConfig()
        .setProperty("service-dns", "hazelcast-service.default.svc.cluster.local")
        .setProperty("use-public-ip", "true")
        .setProperty("service-per-pod-label", "statefulset.kubernetes.io/pod-name");
    
    // 分区组配置
    PartitionGroupConfig partitionGroupConfig = config.getPartitionGroupConfig();
    partitionGroupConfig.setEnabled(true)
                       .setGroupType(PartitionGroupConfig.MemberGroupType.ZONE_AWARE);
    
    return config;
}

7.2 持久化存储

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-springboot-app
spec:
  serviceName: hazelcast-service
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - name: hazelcast-persistent-storage
          mountPath: /data/hazelcast
  volumeClaimTemplates:
  - metadata:
      name: hazelcast-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi

8. 监控和告警

8.1 Prometheus 配置

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'hazelcast'
        metrics_path: '/metrics'
        static_configs:
          - targets: ['my-springboot-app:8080']  # Hazelcast metrics endpoint

8.2 Grafana 仪表板

{
  "panels": [
    {
      "type": "graph",
      "title": "Hazelcast Cluster Size",
      "targets": [{
        "expr": "hazelcast_cluster_members_count"
      }]
    },
    {
      "type": "graph",
      "title": "Map Operations",
      "targets": [{
        "expr": "hazelcast_map_puts_total"
      }, {
        "expr": "hazelcast_map_gets_total"
      }]
    }
  ]
}

9. 故障排除指南

9.1 常见问题及解决方案

问题 症状 解决方案
节点无法发现 日志显示 "No other member is found" 1. 检查 RBAC 权限<br>2. 验证服务 DNS 名称<br>3. 确认网络策略允许 5701 端口通信
集群分裂 多个小集群同时存在 1. 设置 
"hazelcast.initial.min.cluster.size"<br>2. 配置分区组策略
内存不足 Pod 频繁重启 1. 增加内存限制<br>2. 配置堆外内存<br>3. 启用持久化存储
序列化错误 ClassCastException 1. 使用 IdentifiedDataSerializable<br>2. 配置自定义序列化器

9.2 诊断命令

# 检查 Hazelcast 集群状态
kubectl exec <pod-name> -- curl http://localhost:5701/hazelcast/health/cluster-state

# 获取成员列表
kubectl exec <pod-name> -- curl http://localhost:5701/hazelcast/health/cluster-members

# 检查网络连接
kubectl exec <pod-name> -- nc -zv hazelcast-service.default.svc.cluster.local 5701

10. 性能优化建议

10.1 JVM 参数优化

# Dockerfile 片段
ENV JAVA_OPTS="-XX:+UseG1GC \
               -XX:MaxGCPauseMillis=200 \
               -XX:InitiatingHeapOccupancyPercent=35 \
               -XX:+ParallelRefProcEnabled \
               -XX:+PerfDisableSharedMem \
               -XX:+AlwaysPreTouch \
               -Xms2g -Xmx2g \
               -Dhazelcast.jmx=true"

10.2 Hazelcast 配置优化

@Bean
public Config hazelcastConfiguration() {
    Config config = new Config();
    
    // 优化网络缓冲区
    config.setProperty("hazelcast.socket.receive.buffer.size", "64")
          .setProperty("hazelcast.socket.send.buffer.size", "64");
    
    // 优化线程池
    config.getExecutorConfig("default")
          .setPoolSize(16)
          .setQueueCapacity(10000);
    
    // 启用本地内存
    config.getNativeMemoryConfig()
          .setEnabled(true)
          .setSize(new MemorySize(2, MemoryUnit.GIGABYTES));
    
    return config;
}

总结

在 GKE 上嵌入 Hazelcast 到 Spring Boot 应用的关键步骤：

1. 依赖管理：添加必要的 Hazelcast 依赖
2. 配置 Hazelcast：使用 Kubernetes 发现机制
3. Kubernetes 部署：
   - 开放 5701 端口
   - 配置 Headless Service
   - 设置 RBAC 权限
4. 网络配置：
   - 网络策略允许 Pod 间通信
   - GKE 防火墙规则
5. 持久化：为有状态应用配置持久卷
6. 监控：集成 Prometheus 和 Grafana

通过这种架构，Hazelcast 集群能够：

- 自动发现并加入集群
- 在 Pod 重启或扩展时保持数据一致性
- 提供高性能的分布式数据存储
- 无缝集成 Spring Boot 缓存抽象

生产环境部署建议：

- 至少 3 个节点保证高可用
- 使用区域感知部署提高容错性
- 配置资源请求和限制
- 实施定期备份策略
- 启用详细监控和告警
