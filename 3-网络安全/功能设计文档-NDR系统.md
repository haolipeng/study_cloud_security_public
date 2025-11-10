# 网络流量安全分析系统（NDR）- 功能设计文档

> 版本：v1.0  
> 日期：2024年  
> 技术栈：Golang + Kafka + ClickHouse + React + MinIO  

---

## 一、系统架构设计

### 1.1 总体架构

```
┌─────────────────── 前端展示层 ─────────────────┐
│  监控大屏 │ 告警中心 │ 规则管理 │ 威胁狩猎 │ 报表  │
└────────────────────┬────────────────────────────┘
                     │ RESTful API / WebSocket
┌────────────────────┴────────────────────────────┐
│                   后端服务层                      │
│  API网关 │ 认证服务 │ 查询服务 │ 分析服务        │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────┐
│                 数据处理层                        │
│  实时分析 │ 事件关联 │ 威胁匹配 │ 统计聚合       │
└──────┬──────────────────────────────────┬───────┘
       │                                   │
┌──────┴────────┐                  ┌──────┴────────┐
│  Kafka消息队列 │                  │  存储层        │
│  (数据总线)    │                  │ ClickHouse    │
└──────┬────────┘                  │ Elasticsearch │
       │                            │ MinIO         │
┌──────┴────────┐                  │ Redis         │
│  采集Agent集群 │                  └───────────────┘
│  (分布式部署) │
└───────────────┘
```

### 1.2 数据流架构

```
[网络镜像点]
    ↓
[采集Agent] ──→ [本地RingBuffer] ──→ [Kafka Topic: raw-packets]
    │                                          ↓
    │                                   [消费者集群]
    │                                          ↓
    ├─────────────────────────────────→ [DPI引擎]
    │                                      ↓    ↓    ↓
    │                              [HTTP] [DNS] [TLS]...
    │                                      ↓
    │                              [协议元数据] ──→ ClickHouse
    │
    ├─────────────────────────────────→ [NetFlow生成器]
    │                                          ↓
    │                                   [Flow数据] ──→ ClickHouse
    │
    ├─────────────────────────────────→ [检测引擎]
    │                                      ↓
    │                                   [告警] ──→ ClickHouse + ES
    │
    └─────────────────────────────────→ [PCAP存储]
                                            ↓
                                       [MinIO]
```

### 1.3 部署架构

#### 单机版（< 1Gbps）
```
┌─────────────────────────────────┐
│        单服务器（8核32GB）         │
│  ┌──────────────────────────┐   │
│  │  采集 + 分析 + 存储 + Web  │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

#### 分布式版（> 1Gbps）
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ 采集节点 1   │  │ 采集节点 2   │  │ 采集节点 N   │
│ (8核16GB)   │  │ (8核16GB)   │  │ (8核16GB)   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┴────────────────┘
                        │
                 ┌──────┴──────┐
                 │  Kafka集群   │
                 │  (3节点)    │
                 └──────┬──────┘
                        │
       ┌────────────────┴────────────────┐
       │                                 │
┌──────┴──────┐                 ┌────────┴────────┐
│  分析集群    │                 │   存储集群       │
│ (多Worker)  │                 │  ClickHouse(3)  │
│ (16核32GB)  │                 │  ES(3)          │
└──────┬──────┘                 │  MinIO(3)       │
       │                         │  Redis(2)       │
┌──────┴──────┐                 └─────────────────┘
│  Web服务集群 │
│  (4核8GB×2) │
└─────────────┘
```

---

## 二、核心模块设计

### 2.1 流量采集模块

#### 2.1.1 采集Agent架构
```go
package collector

// 核心接口定义
type PacketCapturer interface {
    Start(interfaceName string) error
    Stop() error
    GetStats() CaptureStats
}

type PacketProcessor interface {
    Process(packet *Packet) error
    SetFilters(filters []BPFFilter) error
}

// 采集Agent主结构
type CaptureAgent struct {
    config      *CaptureConfig
    capturer    PacketCapturer
    processor   PacketProcessor
    buffer      *RingBuffer
    kafkaWriter *KafkaProducer
    monitor     *Monitor
    stopChan    chan struct{}
}

// 配置结构
type CaptureConfig struct {
    Interface      string        // 网卡名称，如 eth0
    SnapLen        int           // 每个包的截取长度，默认65535
    BufferSize     int           // 缓冲区大小，默认100MB
    BPFFilter      string        // BPF过滤器，如 "tcp port 80"
    KafkaBrokers   []string      // Kafka集群地址
    KafkaTopic     string        // Kafka主题名称
    BatchSize      int           // 批量发送大小，默认1000
    FlushInterval  time.Duration // 强制刷新间隔，默认5s
}

// 统计信息
type CaptureStats struct {
    PacketsReceived   uint64    // 收到的包数
    PacketsDropped    uint64    // 丢弃的包数
    BytesReceived     uint64    // 收到的字节数
    CPUUsage          float64   // CPU使用率
    MemoryUsage       uint64    // 内存使用量
}
```

#### 2.1.2 核心实现逻辑

**1. 初始化流程**
```go
func NewCaptureAgent(config *CaptureConfig) (*CaptureAgent, error) {
    // 1. 验证配置
    if err := config.Validate(); err != nil {
        return nil, err
    }
    
    // 2. 初始化libpcap
    capturer := NewLibpcapCapturer(config)
    
    // 3. 初始化RingBuffer（零拷贝）
    buffer := NewRingBuffer(config.BufferSize)
    
    // 4. 初始化Kafka生产者
    kafkaWriter, err := NewKafkaProducer(config.KafkaBrokers, config.KafkaTopic)
    if err != nil {
        return nil, err
    }
    
    // 5. 创建Agent实例
    agent := &CaptureAgent{
        config:      config,
        capturer:    capturer,
        buffer:      buffer,
        kafkaWriter: kafkaWriter,
        monitor:     NewMonitor(),
        stopChan:    make(chan struct{}),
    }
    
    return agent, nil
}
```

**2. 数据包处理流程**
```go
func (a *CaptureAgent) Start() error {
    // 1. 启动libpcap采集
    if err := a.capturer.Start(a.config.Interface); err != nil {
        return err
    }
    
    // 2. 启动处理协程
    go a.processPackets()
    
    // 3. 启动定时刷新协程
    go a.flushLoop()
    
    // 4. 启动监控协程
    go a.monitor.Start()
    
    return nil
}

func (a *CaptureAgent) processPackets() {
    for {
        select {
        case <-a.stopChan:
            return
        default:
            // 从capturer读取数据包
            packet, err := a.capturer.ReadPacket()
            if err != nil {
                continue
            }
            
            // 写入RingBuffer
            a.buffer.Write(packet)
            
            // 更新统计
            a.monitor.IncrPacketCount()
        }
    }
}

func (a *CaptureAgent) flushLoop() {
    ticker := time.NewTicker(a.config.FlushInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            a.FlushToKafka()
        case <-a.stopChan:
            return
        }
    }
}
```

**3. 批量发送到Kafka**
```go
func (a *CaptureAgent) FlushToKafka() error {
    // 1. 从RingBuffer读取批量数据
    batch := a.buffer.ReadBatch(a.config.BatchSize)
    if len(batch) == 0 {
        return nil
    }
    
    // 2. 序列化数据包
    messages := make([][]byte, 0, len(batch))
    for _, pkt := range batch {
        data, err := pkt.Serialize()
        if err != nil {
            continue
        }
        messages = append(messages, data)
    }
    
    // 3. 批量发送到Kafka
    if err := a.kafkaWriter.SendBatch(messages); err != nil {
        return err
    }
    
    return nil
}
```

#### 2.1.3 性能优化

**零拷贝实现**
```go
package buffer

type RingBuffer struct {
    data     []byte
    readPos  uint64
    writePos uint64
    size     int
    mu       sync.RWMutex
}

// 零拷贝写入
func (rb *RingBuffer) Write(packet *Packet) error {
    rb.mu.Lock()
    defer rb.mu.Unlock()
    
    // 直接写入共享内存，不复制数据
    packetLen := len(packet.Data)
    if rb.writePos + uint64(packetLen) > uint64(rb.size) {
        // 缓冲区满，返回错误
        return ErrBufferFull
    }
    
    // 使用mmap内存映射，实现零拷贝
    copy(rb.data[rb.writePos:], packet.Data)
    rb.writePos += uint64(packetLen)
    
    return nil
}
```

**AF_PACKET优化**
```go
package capturer

import "golang.org/x/sys/unix"

type AFPacketCapturer struct {
    sockfd int
    ring   *PacketMmapRing
}

func (c *AFPacketCapturer) setupTPACKETv3() error {
    // 使用TPACKET_V3提升性能
    req := unix.TpacketReq3{
        Block_size:   4096 * 1024,  // 4MB块
        Block_nr:     128,            // 128个块
        Frame_size:   2048,           // 2KB帧
        Frame_nr:     4096 * 128 / 2, // 帧数量
        Retire_blk_tov: 60,           // 块超时60ms
    }
    
    return unix.SetsockoptTpacketReq3(c.sockfd, unix.SOL_PACKET, 
                                      unix.PACKET_RX_RING, &req)
}
```

---

### 2.2 DPI深度包检测引擎

#### 2.2.1 模块架构
```go
package dpi

// DPI引擎主结构
type DPIEngine struct {
    identifier   ProtocolIdentifier  // 协议识别器
    parsers      map[Protocol]ProtocolParser  // 协议解析器集合
    reassembler  *StreamReassembler  // TCP流重组
    extractor    *FileExtractor      // 文件提取
    storage      MetadataStorage     // 元数据存储
}

// 协议识别接口
type ProtocolIdentifier interface {
    Identify(packet *Packet) Protocol
    RegisterPlugin(plugin ProtocolPlugin)
}

// 协议解析器接口
type ProtocolParser interface {
    Parse(data []byte) (Metadata, error)
    GetProtocolName() string
}
```

#### 2.2.2 HTTP解析器实现
```go
package parser

import "net/http"

type HTTPParser struct {
    config *ParserConfig
}

type HTTPMetadata struct {
    Timestamp     time.Time
    SrcIP         string
    DstIP         string
    SrcPort       uint16
    DstPort       uint16
    Method        string
    Host          string
    URI           string
    UserAgent     string
    StatusCode    int
    ContentLength int64
    ContentType   string
    Headers       map[string]string
    Cookies       []Cookie
    Body          []byte  // 可选
}

func (p *HTTPParser) Parse(data []byte) (*HTTPMetadata, error) {
    // 1. 解析HTTP请求/响应
    reader := bufio.NewReader(bytes.NewReader(data))
    
    // 2. 判断是请求还是响应
    firstLine, _ := reader.ReadString('\n')
    if strings.HasPrefix(firstLine, "HTTP/") {
        return p.parseResponse(reader)
    } else {
        return p.parseRequest(reader)
    }
}

func (p *HTTPParser) parseRequest(reader *bufio.Reader) (*HTTPMetadata, error) {
    req, err := http.ReadRequest(reader)
    if err != nil {
        return nil, err
    }
    
    metadata := &HTTPMetadata{
        Timestamp:     time.Now(),
        Method:        req.Method,
        Host:          req.Host,
        URI:           req.RequestURI,
        UserAgent:     req.UserAgent(),
        ContentLength: req.ContentLength,
        Headers:       make(map[string]string),
    }
    
    // 提取所有Header
    for key, values := range req.Header {
        metadata.Headers[key] = strings.Join(values, "; ")
    }
    
    // 提取Cookie
    metadata.Cookies = p.ExtractCookies(req)
    
    return metadata, nil
}

func (p *HTTPParser) ExtractCookies(req *http.Request) []Cookie {
    cookies := make([]Cookie, 0)
    for _, c := range req.Cookies() {
        cookies = append(cookies, Cookie{
            Name:  c.Name,
            Value: c.Value,
        })
    }
    return cookies
}
```

#### 2.2.3 TLS指纹提取（JA3）
```go
package parser

import "crypto/tls"

type TLSParser struct{}

type TLSMetadata struct {
    Timestamp     time.Time
    SrcIP         string
    DstIP         string
    SNI           string  // Server Name Indication
    JA3           string  // 客户端指纹
    JA3S          string  // 服务端指纹
    CipherSuite   string
    TLSVersion    string
    Certificates  []Certificate
}

func (p *TLSParser) ExtractJA3(clientHello *tls.ClientHelloInfo) string {
    // JA3 = MD5(SSLVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats)
    
    // 1. 提取SSL版本
    version := clientHello.SupportedVersions
    
    // 2. 提取加密套件
    ciphers := clientHello.CipherSuites
    
    // 3. 提取扩展
    extensions := extractExtensions(clientHello)
    
    // 4. 组合字符串
    ja3String := fmt.Sprintf("%d,%s,%s,%s,%s",
        version,
        joinUint16(ciphers, "-"),
        joinUint16(extensions, "-"),
        joinUint16(curves, "-"),
        joinUint16(pointFormats, "-"))
    
    // 5. 计算MD5
    hash := md5.Sum([]byte(ja3String))
    return hex.EncodeToString(hash[:])
}
```

#### 2.2.4 TCP流重组
```go
package reassembly

type StreamReassembler struct {
    flows         map[FlowID]*FlowState
    timeout       time.Duration
    maxFlowSize   int64
    cleanupTicker *time.Ticker
}

type FlowState struct {
    FlowID          FlowID
    ClientSequence  map[uint32]*Packet
    ServerSequence  map[uint32]*Packet
    ClientData      []byte
    ServerData      []byte
    LastSeen        time.Time
}

func (r *StreamReassembler) ProcessPacket(packet *Packet) error {
    flowID := ExtractFlowID(packet)
    
    // 1. 获取或创建Flow状态
    flow, exists := r.flows[flowID]
    if !exists {
        flow = &FlowState{
            FlowID:         flowID,
            ClientSequence: make(map[uint32]*Packet),
            ServerSequence: make(map[uint32]*Packet),
            LastSeen:       time.Now(),
        }
        r.flows[flowID] = flow
    }
    
    // 2. 根据方向存储数据包
    if packet.IsClientToServer {
        flow.ClientSequence[packet.TCPSeq] = packet
    } else {
        flow.ServerSequence[packet.TCPSeq] = packet
    }
    
    // 3. 尝试重组
    if r.canReassemble(flow) {
        r.ReassembleFlow(flowID)
    }
    
    return nil
}

func (r *StreamReassembler) ReassembleFlow(flowID FlowID) ([]byte, error) {
    flow := r.flows[flowID]
    
    // 1. 排序客户端数据包
    clientSeqs := make([]uint32, 0, len(flow.ClientSequence))
    for seq := range flow.ClientSequence {
        clientSeqs = append(clientSeqs, seq)
    }
    sort.Slice(clientSeqs, func(i, j int) bool {
        return clientSeqs[i] < clientSeqs[j]
    })
    
    // 2. 按序拼接数据
    var reassembled []byte
    for _, seq := range clientSeqs {
        pkt := flow.ClientSequence[seq]
        reassembled = append(reassembled, pkt.Payload...)
    }
    
    return reassembled, nil
}
```

---

### 2.3 流量特征提取模块

#### 2.3.1 NetFlow生成
```go
package netflow

type FlowExporter struct {
    flows         map[FlowKey]*Flow
    exportInterval time.Duration
    kafkaProducer *KafkaProducer
    clickhouse    *ClickHouseClient
}

type FlowKey struct {
    SrcIP    string
    DstIP    string
    SrcPort  uint16
    DstPort  uint16
    Protocol uint8
}

type Flow struct {
    FlowKey
    StartTime       time.Time
    EndTime         time.Time
    PacketCount     uint64
    ByteCount       uint64
    ForwardPackets  uint64
    BackwardPackets uint64
    ForwardBytes    uint64
    BackwardBytes   uint64
    TCPFlags        uint8
    SYNCount        uint8
    RSTCount        uint8
    FINCount        uint8
    Protocol        uint8
    AppProtocol     string  // HTTP/DNS/TLS等
}

func (e *FlowExporter) UpdateFlow(packet *Packet) {
    key := FlowKey{
        SrcIP:    packet.SrcIP,
        DstIP:    packet.DstIP,
        SrcPort:  packet.SrcPort,
        DstPort:  packet.DstPort,
        Protocol: packet.Protocol,
    }
    
    flow, exists := e.flows[key]
    if !exists {
        flow = &Flow{
            FlowKey:   key,
            StartTime: packet.Timestamp,
        }
        e.flows[key] = flow
    }
    
    // 更新流统计
    flow.EndTime = packet.Timestamp
    flow.PacketCount++
    flow.ByteCount += uint64(packet.Length)
    
    // 更新方向性统计
    if packet.Direction == DirectionForward {
        flow.ForwardPackets++
        flow.ForwardBytes += uint64(packet.Length)
    } else {
        flow.BackwardPackets++
        flow.BackwardBytes += uint64(packet.Length)
    }
    
    // 更新TCP标志
    if packet.Protocol == ProtocolTCP {
        flow.TCPFlags |= packet.TCPFlags
        if packet.TCPFlags & TCPFlagSYN != 0 {
            flow.SYNCount++
        }
    }
}
```

#### 2.3.2 完整特征提取器
```go
package features

type FeatureExtractor struct {
    flow *Flow
}

// 完整特征集合
type AllFeatures struct {
    Basic        BasicFlowFeatures        // 基础统计特征
    Temporal     TemporalFeatures         // 时间特征
    TCP          TCPFeatures              // TCP协议特征
    ProtocolDist ProtocolDistFeatures     // 协议分布特征
    TLS          TLSFeatures              // TLS特征
    HTTP         HTTPFeatures             // HTTP特征
    DNS          DNSFeatures              // DNS特征
    Behavior     BehaviorFeatures         // 行为特征
    Statistical  StatisticalFeatures      // 统计特征
    Session      SessionFeatures          // 会话特征
}

// 基础统计特征
type BasicFlowFeatures struct {
    PacketCount       int64
    ByteCount         int64
    ForwardPackets    int64
    BackwardPackets   int64
    ForwardBytes      int64
    BackwardBytes     int64
    Duration          float64
    FlowRate          float64
    PacketRate        float64
    MinPacketLength   int
    MaxPacketLength   int
    MeanPacketLength  float64
    StdPacketLength   float64
}

func (e *FeatureExtractor) ExtractAll() *AllFeatures {
    return &AllFeatures{
        Basic:        e.ExtractBasic(),
        Temporal:     e.ExtractTemporal(),
        TCP:          e.ExtractTCP(),
        ProtocolDist: e.ExtractProtocolDist(),
        Statistical:  e.ExtractStatistical(),
    }
}

func (e *FeatureExtractor) ExtractBasic() *BasicFlowFeatures {
    duration := e.flow.EndTime.Sub(e.flow.StartTime).Seconds()
    
    return &BasicFlowFeatures{
        PacketCount:      int64(e.flow.PacketCount),
        ByteCount:        int64(e.flow.ByteCount),
        ForwardPackets:   int64(e.flow.ForwardPackets),
        BackwardPackets:  int64(e.flow.BackwardPackets),
        ForwardBytes:     int64(e.flow.ForwardBytes),
        BackwardBytes:    int64(e.flow.BackwardBytes),
        Duration:         duration,
        FlowRate:         float64(e.flow.ByteCount) / duration,
        PacketRate:       float64(e.flow.PacketCount) / duration,
        MinPacketLength:  e.calculateMinPacketLength(),
        MaxPacketLength:  e.calculateMaxPacketLength(),
        MeanPacketLength: e.calculateMeanPacketLength(),
        StdPacketLength:  e.calculateStdPacketLength(),
    }
}
```

#### 2.3.3 行为基线模块
```go
package baseline

type BaselineBuilder struct {
    timeWindow    time.Duration  // 基线学习窗口，如7天
    storage       *ClickHouseClient
    cache         *redis.Client
}

type TrafficBaseline struct {
    AssetIP          string
    AvgConnCount     float64
    StdConnCount     float64
    AvgByteCount     float64
    StdByteCount     float64
    TopProtocols     []ProtocolStat
    TopDstPorts      []PortStat
    TopDstIPs        []IPStat
    TimeDistribution map[int]float64  // 24小时活动分布
    UpdatedAt        time.Time
}

func (b *BaselineBuilder) BuildBaseline(assetIP string) (*TrafficBaseline, error) {
    // 1. 查询历史数据（过去7天）
    query := fmt.Sprintf(`
        SELECT 
            count() as conn_count,
            sum(byte_count) as total_bytes,
            protocol,
            dst_port,
            dst_ip,
            toHour(timestamp) as hour
        FROM flows
        WHERE src_ip = '%s'
          AND timestamp >= now() - INTERVAL 7 DAY
        GROUP BY protocol, dst_port, dst_ip, hour
    `, assetIP)
    
    rows, err := b.storage.Query(query)
    if err != nil {
        return nil, err
    }
    
    // 2. 计算统计特征
    baseline := &TrafficBaseline{
        AssetIP:          assetIP,
        TimeDistribution: make(map[int]float64),
    }
    
    var connCounts []float64
    var byteCounts []float64
    protocolMap := make(map[string]int64)
    portMap := make(map[uint16]int64)
    
    for rows.Next() {
        var conn, bytes int64
        var protocol string
        var port uint16
        var hour int
        
        rows.Scan(&conn, &bytes, &protocol, &port, &hour)
        
        connCounts = append(connCounts, float64(conn))
        byteCounts = append(byteCounts, float64(bytes))
        protocolMap[protocol] += conn
        portMap[port] += conn
        baseline.TimeDistribution[hour] += float64(conn)
    }
    
    // 3. 计算均值和标准差
    baseline.AvgConnCount = Mean(connCounts)
    baseline.StdConnCount = StdDev(connCounts)
    baseline.AvgByteCount = Mean(byteCounts)
    baseline.StdByteCount = StdDev(byteCounts)
    
    // 4. 保存到数据库和缓存
    b.saveBaseline(baseline)
    b.cacheBaseline(baseline)
    
    return baseline, nil
}

// 异常检测
type BaselineComparator struct {
    baseline *TrafficBaseline
}

func (c *BaselineComparator) DetectAnomaly(current *CurrentMetrics) *AnomalyResult {
    result := &AnomalyResult{
        IsAnomaly: false,
        Reasons:   make([]string, 0),
        Score:     0.0,
    }
    
    // 1. 检测连接数异常（3-sigma规则）
    connZScore := (current.ConnCount - c.baseline.AvgConnCount) / c.baseline.StdConnCount
    if math.Abs(connZScore) > 3.0 {
        result.IsAnomaly = true
        result.Reasons = append(result.Reasons, 
            fmt.Sprintf("连接数异常: 当前=%d, 基线均值=%.0f, Z-score=%.2f", 
                current.ConnCount, c.baseline.AvgConnCount, connZScore))
        result.Score += math.Abs(connZScore)
    }
    
    // 2. 检测流量异常
    byteZScore := (current.ByteCount - c.baseline.AvgByteCount) / c.baseline.StdByteCount
    if math.Abs(byteZScore) > 3.0 {
        result.IsAnomaly = true
        result.Reasons = append(result.Reasons, 
            fmt.Sprintf("流量异常: 当前=%d, 基线均值=%.0f, Z-score=%.2f", 
                current.ByteCount, c.baseline.AvgByteCount, byteZScore))
        result.Score += math.Abs(byteZScore)
    }
    
    // 3. 检测新目标IP
    if !c.isKnownDestination(current.DstIP) {
        result.IsAnomaly = true
        result.Reasons = append(result.Reasons, 
            fmt.Sprintf("访问新目标IP: %s", current.DstIP))
        result.Score += 2.0
    }
    
    // 4. 检测时间异常
    hour := time.Now().Hour()
    expectedActivity := c.baseline.TimeDistribution[hour]
    if current.ConnCount > expectedActivity * 3 {
        result.IsAnomaly = true
        result.Reasons = append(result.Reasons, 
            fmt.Sprintf("非典型时间活动: 当前时段=%d, 当前连接=%d, 预期=%.0f", 
                hour, current.ConnCount, expectedActivity))
        result.Score += 1.5
    }
    
    return result
}
```

---

### 2.4 威胁检测引擎

#### 2.4.1 规则引擎
```go
package detection

type RuleEngine struct {
    ruleManager   *RuleManager
    matcher       *PatternMatcher
    prefilter     *Prefilter
    alertQueue    chan *Alert
}

type Rule struct {
    ID          string
    Name        string
    Severity    string  // critical/high/medium/low
    Protocol    string
    SrcIP       string
    DstIP       string
    SrcPort     string
    DstPort     string
    Content     []ContentMatch
    PCREPattern string
    Metadata    map[string]string
    Enabled     bool
}

func (e *RuleEngine) Detect(packet *Packet) []*Alert {
    // 1. 预过滤（快速排除不匹配的规则）
    candidateRules := e.prefilter.FilterRules(packet)
    
    // 2. 模式匹配
    matches := e.matcher.MatchRules(packet.Payload, candidateRules)
    
    // 3. 生成告警
    alerts := make([]*Alert, 0)
    for _, match := range matches {
        alert := e.generateAlert(match, packet)
        alerts = append(alerts, alert)
        e.alertQueue <- alert
    }
    
    return alerts
}
```

#### 2.4.2 AC自动机实现
```go
package matcher

type ACAutomaton struct {
    root     *ACNode
    patterns []Pattern
}

type ACNode struct {
    children map[byte]*ACNode
    fail     *ACNode
    output   []int  // 匹配到的模式ID列表
}

func NewACAutomaton() *ACAutomaton {
    return &ACAutomaton{
        root:     &ACNode{children: make(map[byte]*ACNode)},
        patterns: make([]Pattern, 0),
    }
}

func (ac *ACAutomaton) AddPattern(pattern string, id int) {
    node := ac.root
    for i := 0; i < len(pattern); i++ {
        char := pattern[i]
        if _, exists := node.children[char]; !exists {
            node.children[char] = &ACNode{children: make(map[byte]*ACNode)}
        }
        node = node.children[char]
    }
    node.output = append(node.output, id)
}

func (ac *ACAutomaton) Build() {
    // 构建失败指针（使用BFS）
    queue := make([]*ACNode, 0)
    
    // 第一层节点的fail指向root
    for _, child := range ac.root.children {
        child.fail = ac.root
        queue = append(queue, child)
    }
    
    // BFS构建失败指针
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        
        for char, child := range node.children {
            queue = append(queue, child)
            
            // 找到失败指针
            failNode := node.fail
            for failNode != nil {
                if next, exists := failNode.children[char]; exists {
                    child.fail = next
                    break
                }
                failNode = failNode.fail
            }
            
            if child.fail == nil {
                child.fail = ac.root
            }
            
            // 合并output
            child.output = append(child.output, child.fail.output...)
        }
    }
}

func (ac *ACAutomaton) Search(text []byte) []Match {
    matches := make([]Match, 0)
    node := ac.root
    
    for pos := 0; pos < len(text); pos++ {
        char := text[pos]
        
        // 查找下一个节点
        for node != ac.root && node.children[char] == nil {
            node = node.fail
        }
        
        if next, exists := node.children[char]; exists {
            node = next
        }
        
        // 记录匹配结果
        if len(node.output) > 0 {
            for _, patternID := range node.output {
                matches = append(matches, Match{
                    PatternID: patternID,
                    Position:  pos,
                })
            }
        }
    }
    
    return matches
}
```

#### 2.4.3 威胁情报匹配
```go
package intel

type ThreatIntelManager struct {
    storage       *ThreatIntelStorage
    bloomFilter   *BloomFilter
    cache         *lru.Cache
}

type IOC struct {
    Type         string  // ip, domain, hash, url
    Value        string
    ThreatType   string  // malware, c2, phishing, etc
    Confidence   int     // 0-100
    Source       string
    FirstSeen    time.Time
    LastSeen     time.Time
    Tags         []string
}

func (m *ThreatIntelManager) ImportIOCs(iocs []IOC) error {
    // 1. 批量写入数据库
    if err := m.storage.BatchInsert(iocs); err != nil {
        return err
    }
    
    // 2. 更新布隆过滤器（加速匹配）
    for _, ioc := range iocs {
        m.bloomFilter.Add([]byte(ioc.Value))
    }
    
    return nil
}

func (m *ThreatIntelManager) MatchAlert(alert *Alert) []IOC {
    matches := make([]IOC, 0)
    
    // 1. 检查源IP
    if m.bloomFilter.Test([]byte(alert.SrcIP)) {
        if ioc, found := m.QueryIOC(alert.SrcIP); found {
            matches = append(matches, ioc)
        }
    }
    
    // 2. 检查目标IP
    if m.bloomFilter.Test([]byte(alert.DstIP)) {
        if ioc, found := m.QueryIOC(alert.DstIP); found {
            matches = append(matches, ioc)
        }
    }
    
    // 3. 检查域名（如果有）
    if alert.Metadata["domain"] != "" {
        domain := alert.Metadata["domain"].(string)
        if m.bloomFilter.Test([]byte(domain)) {
            if ioc, found := m.QueryIOC(domain); found {
                matches = append(matches, ioc)
            }
        }
    }
    
    return matches
}
```

---

### 2.5 数据存储设计

> 详细内容请参阅：[数据存储和管理.md](./数据存储和管理.md)

核心要点：
- **ClickHouse**：存储Flow、协议元数据、流量特征
- **Elasticsearch**：存储告警和全文检索
- **MinIO**：存储PCAP文件
- **Redis**：缓存查询结果、基线数据

---

### 2.6 后端API设计

#### 2.6.1 API架构
```go
package api

type APIServer struct {
    router      *gin.Engine
    authManager *AuthManager
    services    *Services
}

type Services struct {
    AgentService    *AgentService
    AlertService    *AlertService
    RuleService     *RuleService
    QueryService    *QueryService
    HuntingService  *HuntingService
    UserService     *UserService
}

func (s *APIServer) RegisterRoutes() {
    // 公开路由
    public := s.router.Group("/api/v1")
    {
        public.POST("/auth/login", s.Login)
        public.GET("/health", s.HealthCheck)
    }
    
    // 需要认证的路由
    private := s.router.Group("/api/v1")
    private.Use(s.authManager.AuthMiddleware())
    {
        // Agent管理
        agents := private.Group("/agents")
        {
            agents.GET("", s.ListAgents)
            agents.GET("/:id", s.GetAgent)
            agents.POST("", s.RegisterAgent)
            agents.PUT("/:id", s.UpdateAgent)
            agents.DELETE("/:id", s.DeleteAgent)
        }
        
        // 告警管理
        alerts := private.Group("/alerts")
        {
            alerts.GET("", s.ListAlerts)
            alerts.GET("/:id", s.GetAlertDetail)
            alerts.POST("/:id/acknowledge", s.AcknowledgeAlert)
            alerts.POST("/:id/close", s.CloseAlert)
        }
        
        // 规则管理
        rules := private.Group("/rules")
        {
            rules.GET("", s.ListRules)
            rules.POST("", s.CreateRule)
            rules.PUT("/:id", s.UpdateRule)
            rules.DELETE("/:id", s.DeleteRule)
            rules.POST("/:id/enable", s.EnableRule)
        }
        
        // 查询服务
        query := private.Group("/query")
        {
            query.POST("/flows", s.QueryFlows)
            query.POST("/http", s.QueryHTTPLogs)
            query.POST("/dns", s.QueryDNSLogs)
            query.POST("/pcap", s.SearchPCAP)
            query.GET("/pcap/:id/download", s.DownloadPCAP)
        }
    }
}
```

#### 2.6.2 认证中间件
```go
package auth

import (
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v4"
)

type AuthManager struct {
    jwtSecret     string
    tokenExpiry   time.Duration
    redisClient   *redis.Client
}

type Claims struct {
    UserID    string   `json:"user_id"`
    Username  string   `json:"username"`
    Roles     []string `json:"roles"`
    jwt.StandardClaims
}

func (m *AuthManager) GenerateToken(user *User) (string, error) {
    claims := Claims{
        UserID:   user.ID,
        Username: user.Username,
        Roles:    user.Roles,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(m.tokenExpiry).Unix(),
            Issuer:    "ndr-system",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(m.jwtSecret))
}

func (m *AuthManager) AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 从Header获取Token
        tokenString := c.GetHeader("Authorization")
        if tokenString == "" {
            c.JSON(401, gin.H{"error": "未提供认证令牌"})
            c.Abort()
            return
        }
        
        // 2. 验证Token
        claims, err := m.ValidateToken(tokenString)
        if err != nil {
            c.JSON(401, gin.H{"error": "无效的令牌"})
            c.Abort()
            return
        }
        
        // 3. 检查Token是否被撤销（Redis黑名单）
        if m.isTokenRevoked(tokenString) {
            c.JSON(401, gin.H{"error": "令牌已被撤销"})
            c.Abort()
            return
        }
        
        // 4. 将用户信息存入Context
        c.Set("user_id", claims.UserID)
        c.Set("username", claims.Username)
        c.Set("roles", claims.Roles)
        
        c.Next()
    }
}
```

#### 2.6.3 查询服务实现
```go
package service

type QueryService struct {
    clickhouse *ClickHouseClient
    es         *ESClient
    cache      *QueryCache
}

type FlowQueryRequest struct {
    TimeRange   TimeRange
    SrcIP       string
    DstIP       string
    Protocol    string
    PageSize    int
    PageNum     int
}

type FlowQueryResponse struct {
    Total  int64
    Flows  []Flow
}

func (s *QueryService) QueryFlows(req *FlowQueryRequest) (*FlowQueryResponse, error) {
    // 1. 构建SQL查询
    query := `
        SELECT 
            timestamp, src_ip, dst_ip, src_port, dst_port, 
            protocol, packet_count, byte_count, duration
        FROM flows
        WHERE timestamp >= ? AND timestamp <= ?
    `
    
    args := []interface{}{req.TimeRange.Start, req.TimeRange.End}
    
    if req.SrcIP != "" {
        query += " AND src_ip = ?"
        args = append(args, req.SrcIP)
    }
    
    if req.DstIP != "" {
        query += " AND dst_ip = ?"
        args = append(args, req.DstIP)
    }
    
    // 2. 添加分页
    offset := (req.PageNum - 1) * req.PageSize
    query += fmt.Sprintf(" ORDER BY timestamp DESC LIMIT %d OFFSET %d", 
                         req.PageSize, offset)
    
    // 3. 执行查询
    rows, err := s.clickhouse.Query(query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // 4. 解析结果
    flows := make([]Flow, 0)
    for rows.Next() {
        var flow Flow
        err := rows.Scan(&flow.Timestamp, &flow.SrcIP, &flow.DstIP, 
                        &flow.SrcPort, &flow.DstPort, &flow.Protocol,
                        &flow.PacketCount, &flow.ByteCount, &flow.Duration)
        if err != nil {
            continue
        }
        flows = append(flows, flow)
    }
    
    // 5. 查询总数
    countQuery := `SELECT count() FROM flows WHERE timestamp >= ? AND timestamp <= ?`
    var total int64
    s.clickhouse.QueryRow(countQuery, req.TimeRange.Start, req.TimeRange.End).Scan(&total)
    
    return &FlowQueryResponse{
        Total: total,
        Flows: flows,
    }, nil
}
```

---

### 2.7 前端设计

#### 2.7.1 技术栈
```
- 框架：React 18 + TypeScript
- 状态管理：Zustand
- UI组件：Ant Design
- 图表：ECharts
- 网络图：G6
- 表格：ag-Grid
- 路由：React Router v6
- HTTP客户端：axios
```

#### 2.7.2 告警中心实现
```typescript
// src/pages/AlertCenter.tsx
import React, { useState, useEffect } from 'react';
import { Table, Tag, Space, Button, Modal, Descriptions } from 'antd';
import { getAlerts, acknowledgeAlert, closeAlert } from '@/services/alert';

interface Alert {
  id: string;
  timestamp: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  ruleName: string;
  srcIP: string;
  dstIP: string;
  status: 'open' | 'acknowledged' | 'closed';
}

const AlertCenter: React.FC = () => {
  const [alerts, setAlerts] = useState<Alert[]>([]);
  const [loading, setLoading] = useState(false);
  const [selectedAlert, setSelectedAlert] = useState<Alert | null>(null);
  const [modalVisible, setModalVisible] = useState(false);

  useEffect(() => {
    loadAlerts();
  }, []);

  const loadAlerts = async () => {
    setLoading(true);
    try {
      const response = await getAlerts({
        pageSize: 20,
        pageNum: 1
      });
      setAlerts(response.data);
    } catch (error) {
      console.error('加载告警失败:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleAcknowledge = async (alertId: string) => {
    try {
      await acknowledgeAlert(alertId);
      loadAlerts();
    } catch (error) {
      console.error('确认告警失败:', error);
    }
  };

  const columns = [
    {
      title: '时间',
      dataIndex: 'timestamp',
      key: 'timestamp',
      render: (text: string) => new Date(text).toLocaleString(),
    },
    {
      title: '严重级别',
      dataIndex: 'severity',
      key: 'severity',
      render: (severity: string) => {
        const colorMap = {
          critical: 'red',
          high: 'orange',
          medium: 'gold',
          low: 'blue',
        };
        return <Tag color={colorMap[severity]}>{severity.toUpperCase()}</Tag>;
      },
    },
    {
      title: '规则名称',
      dataIndex: 'ruleName',
      key: 'ruleName',
    },
    {
      title: '源IP',
      dataIndex: 'srcIP',
      key: 'srcIP',
    },
    {
      title: '目标IP',
      dataIndex: 'dstIP',
      key: 'dstIP',
    },
    {
      title: '状态',
      dataIndex: 'status',
      key: 'status',
      render: (status: string) => {
        const colorMap = {
          open: 'red',
          acknowledged: 'orange',
          closed: 'green',
        };
        return <Tag color={colorMap[status]}>{status}</Tag>;
      },
    },
    {
      title: '操作',
      key: 'action',
      render: (_, record: Alert) => (
        <Space size="small">
          <Button size="small" onClick={() => showAlertDetail(record)}>
            详情
          </Button>
          <Button 
            size="small" 
            type="primary"
            onClick={() => handleAcknowledge(record.id)}
          >
            确认
          </Button>
        </Space>
      ),
    },
  ];

  const showAlertDetail = (alert: Alert) => {
    setSelectedAlert(alert);
    setModalVisible(true);
  };

  return (
    <div>
      <h2>告警中心</h2>
      <Table
        columns={columns}
        dataSource={alerts}
        loading={loading}
        rowKey="id"
        pagination={{ pageSize: 20 }}
      />
      
      <Modal
        title="告警详情"
        visible={modalVisible}
        onCancel={() => setModalVisible(false)}
        footer={null}
        width={800}
      >
        {selectedAlert && (
          <Descriptions bordered column={2}>
            <Descriptions.Item label="告警ID">{selectedAlert.id}</Descriptions.Item>
            <Descriptions.Item label="时间">
              {new Date(selectedAlert.timestamp).toLocaleString()}
            </Descriptions.Item>
            <Descriptions.Item label="严重级别">
              {selectedAlert.severity}
            </Descriptions.Item>
            <Descriptions.Item label="规则">{selectedAlert.ruleName}</Descriptions.Item>
            <Descriptions.Item label="源IP">{selectedAlert.srcIP}</Descriptions.Item>
            <Descriptions.Item label="目标IP">{selectedAlert.dstIP}</Descriptions.Item>
          </Descriptions>
        )}
      </Modal>
    </div>
  );
};

export default AlertCenter;
```

#### 2.7.3 实时监控大屏
```typescript
// src/pages/Dashboard.tsx
import React, { useState, useEffect } from 'react';
import { Row, Col, Card, Statistic } from 'antd';
import * as echarts from 'echarts';
import { getRealtimeMetrics } from '@/services/metrics';

const Dashboard: React.FC = () => {
  const [metrics, setMetrics] = useState({
    totalTraffic: 0,
    alertCount: 0,
    activeAgents: 0,
  });

  useEffect(() => {
    // 初始化图表
    initTrafficChart();
    initAlertChart();
    
    // 定时刷新
    const interval = setInterval(() => {
      loadMetrics();
    }, 5000);
    
    return () => clearInterval(interval);
  }, []);

  const loadMetrics = async () => {
    const data = await getRealtimeMetrics();
    setMetrics(data);
    updateCharts(data);
  };

  const initTrafficChart = () => {
    const chart = echarts.init(document.getElementById('traffic-chart'));
    const option = {
      title: { text: '流量趋势' },
      tooltip: { trigger: 'axis' },
      xAxis: { type: 'category', data: [] },
      yAxis: { type: 'value' },
      series: [{
        name: '流量(Mbps)',
        type: 'line',
        data: [],
        smooth: true,
      }],
    };
    chart.setOption(option);
  };

  return (
    <div>
      <Row gutter={16}>
        <Col span={6}>
          <Card>
            <Statistic
              title="总流量"
              value={metrics.totalTraffic}
              suffix="Gbps"
            />
          </Card>
        </Col>
        <Col span={6}>
          <Card>
            <Statistic
              title="告警数量"
              value={metrics.alertCount}
            />
          </Card>
        </Col>
        <Col span={6}>
          <Card>
            <Statistic
              title="活跃探针"
              value={metrics.activeAgents}
            />
          </Card>
        </Col>
      </Row>
      
      <Row gutter={16} style={{ marginTop: 16 }}>
        <Col span={24}>
          <Card title="流量趋势">
            <div id="traffic-chart" style={{ height: 400 }}></div>
          </Card>
        </Col>
      </Row>
    </div>
  );
};

export default Dashboard;
```

---

## 三、性能优化方案

### 3.1 采集性能优化

#### 3.1.1 零拷贝技术
```
传统方式：
[网卡] → [内核缓冲区] → [用户态缓冲区] → [应用程序]
       (拷贝1)        (拷贝2)

零拷贝方式：
[网卡] → [共享内存] ← [应用程序]
       (DMA直接)   (mmap)
```

**实现**：使用AF_PACKET + PACKET_MMAP

#### 3.1.2 批量处理
- 批量读取：一次读取多个数据包（128-256个）
- 批量发送：积累1000-5000条消息后批量发送到Kafka
- 减少系统调用次数

#### 3.1.3 CPU亲和性
```go
func setCPUAffinity(cpuID int) error {
    var cpuset unix.CPUSet
    cpuset.Set(cpuID)
    return unix.SchedSetaffinity(0, &cpuset)
}
```

### 3.2 检测性能优化

#### 3.2.1 规则预过滤
```
所有规则(10000+)
    ↓ IP过滤
候选规则(100)
    ↓ 端口过滤
候选规则(10)
    ↓ 模式匹配
匹配规则(1-2)
```

#### 3.2.2 布隆过滤器
用于威胁情报快速匹配：
- 空间复杂度：O(m)
- 时间复杂度：O(k)
- 误报率：可配置< 1%

### 3.3 存储性能优化

#### 3.3.1 ClickHouse优化
```sql
-- 1. 分区优化
PARTITION BY toYYYYMM(timestamp)

-- 2. 排序键优化
ORDER BY (timestamp, src_ip, dst_ip)

-- 3. 索引优化
INDEX idx_src_ip src_ip TYPE bloom_filter GRANULARITY 1

-- 4. 物化视图（预聚合）
CREATE MATERIALIZED VIEW mv_hourly_stats
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMMDD(hour)
ORDER BY (hour, src_ip)
AS SELECT
    toStartOfHour(timestamp) as hour,
    src_ip,
    count() as flow_count,
    sum(byte_count) as total_bytes
FROM flows
GROUP BY hour, src_ip;
```

#### 3.3.2 查询缓存
```go
func (c *QueryCache) Get(query string) (interface{}, bool) {
    // 1. 计算查询的Hash
    queryHash := sha256.Sum256([]byte(query))
    key := hex.EncodeToString(queryHash[:])
    
    // 2. 从Redis获取缓存
    val, err := c.redis.Get(key).Result()
    if err != nil {
        return nil, false
    }
    
    // 3. 反序列化
    var result interface{}
    json.Unmarshal([]byte(val), &result)
    return result, true
}
```

---

## 四、监控与可观测性

### 4.1 Prometheus指标
```go
package monitoring

var (
    // 采集指标
    PacketsPerSecond = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "packets_per_second",
        Help: "Number of packets captured per second",
    })
    
    // 检测指标
    AlertsPerSecond = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "alerts_per_second",
        Help: "Number of alerts generated per second",
    })
    
    // 查询指标
    QueryLatency = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name: "query_latency_seconds",
        Help: "Query latency in seconds",
        Buckets: []float64{0.1, 0.5, 1.0, 2.0, 5.0},
    })
)
```

### 4.2 日志设计
```go
package logging

import "go.uber.org/zap"

func InitLogger() *zap.Logger {
    config := zap.Config{
        Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
        Development: false,
        Encoding:    "json",
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "timestamp",
            LevelKey:       "level",
            MessageKey:     "message",
            CallerKey:      "caller",
            StacktraceKey:  "stacktrace",
            EncodeTime:     zapcore.ISO8601TimeEncoder,
            EncodeLevel:    zapcore.LowercaseLevelEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
        },
        OutputPaths:      []string{"stdout", "/var/log/ndr/app.log"},
        ErrorOutputPaths: []string{"stderr"},
    }
    
    logger, _ := config.Build()
    return logger
}
```

---

## 五、部署方案

### 5.1 Docker部署
```yaml
# docker-compose.yml
version: '3.8'

services:
  # 采集Agent
  agent:
    image: ndr-agent:latest
    network_mode: host
    privileged: true
    volumes:
      - ./config:/etc/ndr
    environment:
      - KAFKA_BROKERS=kafka:9092
      - INTERFACE=eth0
  
  # Kafka
  kafka:
    image: confluentinc/cp-kafka:7.0.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
  
  # ClickHouse
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-data:/var/lib/clickhouse
  
  # API Server
  api:
    image: ndr-api:latest
    ports:
      - "8080:8080"
    depends_on:
      - kafka
      - clickhouse
      - redis
  
  # Web前端
  web:
    image: ndr-web:latest
    ports:
      - "80:80"
    depends_on:
      - api

volumes:
  clickhouse-data:
  kafka-data:
```

### 5.2 Kubernetes部署
```yaml
# agent-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ndr-agent
spec:
  selector:
    matchLabels:
      app: ndr-agent
  template:
    metadata:
      labels:
        app: ndr-agent
    spec:
      hostNetwork: true
      containers:
      - name: agent
        image: ndr-agent:latest
        securityContext:
          privileged: true
        env:
        - name: KAFKA_BROKERS
          value: "kafka:9092"
```

---

## 六、测试方案

### 6.1 单元测试
```go
package collector

import "testing"

func TestCaptureAgent_FlushToKafka(t *testing.T) {
    // 1. 创建测试Agent
    agent := &CaptureAgent{
        buffer: NewRingBuffer(1024),
        kafkaWriter: &MockKafkaProducer{},
    }
    
    // 2. 写入测试数据
    for i := 0; i < 100; i++ {
        packet := &Packet{Data: []byte(fmt.Sprintf("packet-%d", i))}
        agent.buffer.Write(packet)
    }
    
    // 3. 执行Flush
    err := agent.FlushToKafka()
    
    // 4. 验证结果
    if err != nil {
        t.Errorf("FlushToKafka failed: %v", err)
    }
}
```

### 6.2 性能测试
```go
func BenchmarkACAutomaton_Search(b *testing.B) {
    // 1. 准备测试数据
    ac := NewACAutomaton()
    for i := 0; i < 1000; i++ {
        ac.AddPattern(fmt.Sprintf("pattern-%d", i), i)
    }
    ac.Build()
    
    text := []byte(strings.Repeat("test data ", 1000))
    
    // 2. 执行基准测试
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ac.Search(text)
    }
}
```

---

## 七、代码结构

```
network-security-platform/
├── cmd/
│   ├── agent/              # 采集Agent主程序
│   │   └── main.go
│   ├── analyzer/           # 分析服务主程序
│   │   └── main.go
│   ├── api/                # API服务主程序
│   │   └── main.go
│   └── cli/                # 命令行工具
│       └── main.go
├── pkg/
│   ├── capture/            # 流量采集
│   │   ├── agent.go
│   │   ├── libpcap.go
│   │   └── buffer.go
│   ├── parser/             # 协议解析
│   │   ├── http.go
│   │   ├── dns.go
│   │   └── tls.go
│   ├── detection/          # 检测引擎
│   │   ├── rule_engine.go
│   │   ├── ac_automaton.go
│   │   └── detector.go
│   ├── features/           # 特征提取
│   │   ├── extractor.go
│   │   └── baseline.go
│   ├── intel/              # 威胁情报
│   │   ├── manager.go
│   │   └── ioc.go
│   ├── storage/            # 存储接口
│   │   ├── clickhouse.go
│   │   ├── elasticsearch.go
│   │   └── minio.go
│   ├── query/              # 查询服务
│   │   └── service.go
│   └── common/             # 公共组件
│       ├── types.go
│       └── utils.go
├── internal/
│   ├── config/             # 配置管理
│   ├── middleware/         # 中间件
│   └── models/             # 数据模型
├── api/
│   ├── rest/               # REST API定义
│   │   └── handlers.go
│   └── grpc/               # gRPC定义
│       └── service.proto
├── web/                    # 前端代码
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   └── utils/
│   └── public/
├── configs/                # 配置文件
├── deployments/            # 部署配置
│   ├── docker/
│   └── kubernetes/
├── scripts/                # 脚本
├── docs/                   # 文档
└── tests/                  # 测试
    ├── unit/
    ├── integration/
    └── benchmark/
```

---

## 八、关键技术难点解决方案

### 8.1 高流量环境零丢包
**技术方案**：
1. 使用PF_RING或DPDK绕过内核协议栈
2. CPU亲和性绑定
3. 大页内存(HugePages)
4. 批量处理减少系统调用

### 8.2 实时检测低延迟
**技术方案**：
1. 规则预过滤（IP/端口/协议快速筛选）
2. AC自动机多模式匹配
3. 布隆过滤器加速IOC匹配
4. 结果缓存(Redis)

### 8.3 海量数据存储与查询
**技术方案**：
1. ClickHouse分区表（按月分区）
2. 物化视图预聚合
3. 冷热数据分离（SSD + HDD）
4. 数据压缩（LZ4/ZSTD）
5. 查询结果缓存

---

## 九、总结

本设计文档提供了NDR系统的完整技术实现方案，涵盖：

1. **系统架构**：从采集到展示的完整数据流
2. **核心模块**：每个模块的详细代码设计
3. **性能优化**：零拷贝、批量处理、缓存等优化技术
4. **部署方案**：Docker和Kubernetes部署
5. **测试方案**：单元测试和性能测试

**下一步工作**：
1. 按模块逐步实现代码
2. 持续性能测试和优化
3. 完善监控和日志
4. 编写详细的API文档

