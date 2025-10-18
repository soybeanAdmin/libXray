# libXray Balancer 配置文档

## 概述

本文档专门介绍 libXray 项目中基于 Xray-core balancer 实现故障转移的配置方法，包括 balancer 配置参数详解和 ping 频率设置。

## 1. Balancer 基础配置

Xray-core 内置负载均衡器可以实现多个 outbound 之间的故障转移。基本配置结构如下：

```json
{
  "routing": {
    "balancers": [
      {
        "tag": "failover-balancer",
        "selector": ["proxy-a", "proxy-b"],
        "strategy": {
          "type": "leastPing"
        }
      }
    ],
    "rules": [
      {
        "type": "field",
        "balancerTag": "failover-balancer"
      }
    ]
  },
  "outbounds": [
    {
      "tag": "proxy-a",
      "protocol": "vless",
      "settings": { /* 线路 A 配置 */ }
    },
    {
      "tag": "proxy-b",
      "protocol": "vless", 
      "settings": { /* 线路 B 配置 */ }
    }
  ]
}
```

## 2. Balancer 参数详解

### 2.1 基本参数

**tag (必需)**
- 类型: string
- 说明: balancer 的唯一标识符，用于在路由规则中引用
- 示例: `"failover-balancer"`

**selector (必需)**
- 类型: string[]
- 说明: outbound 标签数组，指定参与负载均衡的出站连接
- 支持前缀匹配，如 `"proxy-"` 会匹配所有以 `proxy-` 开头的 outbound
- 示例: `["proxy-a", "proxy-b", "proxy-c"]`

**strategy (可选)**
- 类型: object
- 说明: 负载均衡策略配置
- 默认值: `{"type": "random"}`

### 2.2 负载均衡策略

**random**
```json
{
  "strategy": {
    "type": "random"
  }
}
```
- 随机选择一个可用的 outbound
- 适用于简单的负载分散场景
- 无需额外配置

**leastPing** (推荐用于故障转移)
```json
{
  "strategy": {
    "type": "leastPing"
  }
}
```
- 选择延迟最低的 outbound
- 需要配合 observatory 模块使用
- 适用于性能优先的故障转移场景

**leastLoad**
```json
{
  "strategy": {
    "type": "leastLoad"
  }
}
```
- 选择当前负载最低的 outbound
- 需要开启流量统计功能
- 适用于流量均衡场景

## 3. Observatory 健康检查配置

### 3.1 基本配置

Observatory 模块负责定期检测 outbound 的健康状态，为 `leastPing` 策略提供延迟数据：

```json
{
  "observatory": {
    "subjectSelector": ["proxy-a", "proxy-b"],
    "probeUrl": "https://www.google.com/generate_204",
    "probeInterval": "30s",
    "enableConcurrency": false
  }
}
```

### 3.2 Observatory 参数详解

**subjectSelector (必需)**
- 类型: string[]
- 说明: 需要进行健康检查的 outbound 标签数组
- 应与 balancer 的 selector 保持一致

**probeUrl (必需)**
- 类型: string
- 说明: 用于探测的目标 URL
- 推荐使用返回 HTTP 204 状态码的地址
- 常用地址：
  - `"https://www.google.com/generate_204"`
  - `"https://www.cloudflare.com/cdn-cgi/trace"`
  - `"https://httpbin.org/status/204"`

**probeInterval (可选)**
- 类型: string (时间格式)
- 说明: ping 探测的频率间隔
- 默认值: `"1m"` (1分钟)
- 支持的时间单位：
  - `"s"`: 秒
  - `"m"`: 分钟  
  - `"h"`: 小时
- 示例：
  - `"30s"`: 每30秒探测一次
  - `"1m"`: 每1分钟探测一次
  - `"2m30s"`: 每2分30秒探测一次

**enableConcurrency (可选)**
- 类型: boolean
- 说明: 是否启用并发探测
- 默认值: `false` (串行探测)
- `true`: 并发探测，速度更快但资源消耗更大
- `false`: 串行探测，资源消耗较小但速度较慢

## 4. libXray 中的 Ping 实现

### 4.1 代码中的 Ping 超时设置

在 `nodep/measure.go` 中定义了 ping 相关的常量：

```go
const (
    PingDelayTimeout int64 = 11000  // 11秒超时判定
    PingDelayError   int64 = 10000  // 10秒错误判定
)
```

### 4.2 Ping 测试函数

libXray 提供了多种 ping 测试方法：

**HTTP Ping**
```go
func Ping(datDir, configPath string, timeout int, url, proxy string) (int64, error)
```
- `timeout`: HTTP 请求超时时间（秒）
- 执行 HEAD 请求测试延迟

**TCP Ping**  
```go
func PingTCP(datDir, configPath string, timeout int, host string, port int, proxy string) (int64, error)
```
- 测试 TCP 连接建立时间

**代理连接测试**
```go
func Connect(datDir, configPath string, timeout int, targetHost string, targetPort int, proxy string) (int64, error)
```
- 测试通过代理连接目标服务器的延迟

## 5. 完整配置示例

### 5.1 基本故障转移配置

以下是一个完整的故障转移配置示例：

```json
{
  "log": {
    "level": "warning"
  },
  "routing": {
    "balancers": [
      {
        "tag": "failover-balancer",
        "selector": ["proxy-a", "proxy-b"],
        "strategy": {
          "type": "leastPing"
        }
      }
    ],
    "rules": [
      {
        "type": "field",
        "balancerTag": "failover-balancer"
      }
    ]
  },
  "outbounds": [
    {
      "tag": "proxy-a",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "server1.example.com",
            "port": 443,
            "users": [
              {
                "id": "your-uuid-here",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls"
      }
    },
    {
      "tag": "proxy-b",
      "protocol": "vless", 
      "settings": {
        "vnext": [
          {
            "address": "server2.example.com",
            "port": 443,
            "users": [
              {
                "id": "your-uuid-here",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls"
      }
    }
  ],
  "observatory": {
    "subjectSelector": ["proxy-a", "proxy-b"],
    "probeUrl": "https://www.google.com/generate_204",
    "probeInterval": "30s",
    "enableConcurrency": false
  }
}
```

### 5.2 多线路配置示例

```json
{
  "routing": {
    "balancers": [
      {
        "tag": "main-balancer",
        "selector": ["hk-", "sg-", "us-"],
        "strategy": {
          "type": "leastPing"
        }
      }
    ],
    "rules": [
      {
        "type": "field",
        "balancerTag": "main-balancer"
      }
    ]
  },
  "outbounds": [
    {
      "tag": "hk-01",
      "protocol": "vless",
      "settings": { /* 香港节点1配置 */ }
    },
    {
      "tag": "hk-02", 
      "protocol": "vless",
      "settings": { /* 香港节点2配置 */ }
    },
    {
      "tag": "sg-01",
      "protocol": "vless",
      "settings": { /* 新加坡节点1配置 */ }
    },
    {
      "tag": "us-01",
      "protocol": "vless", 
      "settings": { /* 美国节点1配置 */ }
    }
  ],
  "observatory": {
    "subjectSelector": ["hk-", "sg-", "us-"],
    "probeUrl": "https://www.google.com/generate_204",
    "probeInterval": "1m"
  }
}
```

## 6. 关键技术细节

### 6.1 Ping 频率控制

**Observatory 配置中的 probeInterval**
- 这是 Xray-core 自动 ping 的频率设置
- 可以设置为任意时间间隔，如 `"30s"`、`"1m"`、`"2m30s"`
- **默认值**: `"1m"` (如果不设置)

**libXray 手动 Ping**
- libXray 提供的 ping 函数是手动调用的，不是自动的
- 每次调用 `Ping()` 函数会创建新的 Xray 实例进行测试
- 适用于配置验证和单次延迟测试

### 6.2 故障检测机制

**自动故障检测**
- Observatory 模块会定期探测所有 outbound
- 如果某个 outbound 连续探测失败，会被标记为不可用
- leastPing 策略会自动避开不可用的 outbound

**故障恢复**
- 当之前失败的 outbound 恢复正常时，会重新参与负载均衡
- 不需要手动干预，完全自动化

### 6.3 性能优化建议

**probeInterval 设置建议**
- 对于稳定网络: `"1m"` 或 `"2m"`
- 对于不稳定网络: `"30s"`
- 避免设置过短（如 <15s），会增加服务器负担

**enableConcurrency 设置建议**
- 少于5个 outbound: 设置为 `false`
- 5个以上 outbound: 可以设置为 `true`
- 资源有限的设备建议设置为 `false`

## 7. 切换通知机制

### 7.1 默认行为

**重要提醒：Xray-core 的 balancer 故障转移是内部自动进行的，不会直接回调通知到APP层面。**

- balancer 根据策略自动选择最佳 outbound
- 切换过程对上层应用完全透明
- 没有内置的切换事件通知机制

### 7.2 监控切换状态的方法

如果 APP 需要感知线路切换，可以通过以下方式：

**方法1：Stats API 监控**
```go
// 定期查询统计信息
func monitorActiveOutbound() {
    stats, err := xray.QueryStats("http://[::1]:49227/debug/vars")
    if err == nil {
        // 解析统计数据，判断当前活跃的 outbound
        // 通过流量变化推断切换事件
    }
}
```

**方法2：主动健康检查**
```go
// 定期 ping 各个线路
func checkOutboundHealth(outbounds []string) {
    for _, tag := range outbounds {
        delay, _ := xray.Ping(datDir, configPath, 10, testUrl, proxyUrl)
        // 记录延迟变化，推断当前使用的线路
    }
}
```

### 7.3 实现切换通知的示例

```go
type FailoverMonitor struct {
    currentOutbound string
    statsServer     string
    callback        func(oldTag, newTag string)
}

func (m *FailoverMonitor) StartMonitoring() {
    ticker := time.NewTicker(30 * time.Second)
    go func() {
        for range ticker.C {
            active := m.detectActiveOutbound()
            if active != m.currentOutbound && active != "" {
                m.callback(m.currentOutbound, active)
                m.currentOutbound = active
            }
        }
    }()
}

func (m *FailoverMonitor) detectActiveOutbound() string {
    stats, err := xray.QueryStats(m.statsServer)
    if err != nil {
        return ""
    }
    // 解析统计数据，返回当前活跃的 outbound tag
    return parseActiveOutboundFromStats(stats)
}
```

## 8. 使用注意事项

1. **确保 outbound 标签唯一**: balancer 的 selector 中引用的所有 outbound 必须有唯一的 tag
2. **observatory 与 balancer 同步**: observatory 的 subjectSelector 应包含 balancer 中的所有 outbound
3. **probeUrl 可达性**: 确保选择的 probeUrl 在所有 outbound 中都能正常访问
4. **避免过度优化**: 过短的 probeInterval 可能导致频繁切换，影响连接稳定性
5. **切换监控成本**: 如果需要监控切换状态，需要权衡轮询频率和性能开销

## 结论

通过 Xray-core 的 balancer 和 observatory 模块，可以轻松实现自动故障转移功能。关键是正确配置 `probeInterval` 来控制 ping 频率，以及选择合适的负载均衡策略。推荐使用 `leastPing` 策略配合 30秒到1分钟的探测间隔，既能保证故障转移的及时性，又不会造成过多的网络开销。
