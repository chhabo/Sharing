实现步骤
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