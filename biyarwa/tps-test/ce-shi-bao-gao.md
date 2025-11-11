# 测试报告

## 1. 文档介绍

### 1.1. 测试目的

本次测试目的是验证Injective链在测试环境下，bank原生资产转账模块、evm智能合约模块、exchange现货交易模块的性能表现。通过自定义压力测试脚本模拟用户并发场景，采集并分析TPS、节点资源占用率等关键指标。

### 1.2. 测试内容

测试版本：injective v1.16.1测试模块：

1. bank 模块：原生资产（INJ）跨地址转账交易；
2. evm 模块：简单合约部署、ERC20 代币转账交易；
3. exchange 模块：限价单挂单 / 撤单交易；

测试时长：10分钟

### 1.3. 术语和定义

TPS（Transaction Per Second）：每秒成功处理的交易数量；EVM（Ethereum Virtual Machine）：以太坊虚拟机，Injective链EVM模块兼容以太坊生态合约，支持Solidity代码执行；

## 2. 测试环境

### 2.1. 节点部署

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=N2QzYWMwZjMyOTY5ODhiY2Y0MzE4MzZiMmUyYjNkM2FfQUhlTDExRER2TEtFbVhHWWd3dU9KOEV4b3FMMWhhdmpfVG9rZW46SWxVbGI1Q0xFb2ZxMXF4UmdJZWpHU3dicGVnXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

### 2.2. 硬件配置

<table><thead><tr><th width="86">节点序号</th><th>主机ip</th><th>cpu型号</th><th width="100">cpu核数</th><th width="76">内存</th><th width="79">磁盘</th><th>操作系统</th></tr></thead><tbody><tr><td>node1</td><td>118.193.37.73</td><td>Intel Xeon Processor (Cascadelake)</td><td>4</td><td>8G</td><td>100G<br></td><td>Ubuntu 22.04.4 LTS</td></tr><tr><td>node2</td><td>118.193.37.73</td><td>Intel Xeon Processor (Cascadelake)</td><td>4</td><td>8G</td><td>100G<br></td><td>Ubuntu 22.04.4 LTS</td></tr><tr><td>node3</td><td>118.193.37.73</td><td>Intel Xeon Processor (Cascadelake)</td><td>4</td><td>8G</td><td>100G<br></td><td>Ubuntu 22.04.4 LTS</td></tr><tr><td>ndoe4</td><td>118.193.37.73</td><td>Intel Xeon Processor (Cascadelake)</td><td>4</td><td>8G</td><td>100G<br></td><td>Ubuntu 22.04.4 LTS</td></tr></tbody></table>

## 3. 测试方法

### 3.1. API协议

grpc协议用于tps测试

### 3.2. 测试工具

shell脚本，用于部署节点、创建合约等go sdk：用于发送转账交易、统计tps

### 3.3. 测试场景

| 账户数 | 目标账户数 | 统计时长   | 说明            |
| --- | ----- | ------ | ------------- |
| 10  | 1     | 持续10分钟 | 由10个账户向1个账户转账 |

### 3.4. tps统计方法

为准确评估系统的交易处理性能，测试脚本采用自动化统计方式计算 **每分钟及每10分钟的平均 TPS**。

1. 区块高度与时间窗口采样：

脚本每隔 1 分钟记录当前区块高度与时间戳，形成时间区间 `[t, t+Δt]`。

* 当 Δt = 60 秒，用于计算 1 分钟平均 TPS；
* 当 Δt = 600 秒，用于计算 10 分钟平均 TPS。

2. 区块交易数：

在每个时间窗口内，脚本通过节点 gRPC 接口查询区块区间内的所有区块，并统计这些区块中交易数量。

3. TPS 计算：

\$$TPS=ΣTx/Δt\$$其中 Δt 分别为 60 秒或 600 秒。

4. 结果输出：

脚本实时输出每分钟和每10分钟的平均 TPS 值。

## 4. 测试流程

1. 部署节点

```bash
```

2. 初始化账户

```bash
```

3. 原生资产交易
4. 部署合约
5. 合约代币交易
6. 创建市场
7. 现货交易

## 5. 测试结果

### 5.1. CPU & 内存使用情况

#### 5.1.1. bank

内存: 7GCPU单核使用率: 68.8%-120.9%，CPU整体使用率: 86.4%

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=YTY4NDJiMDQ4M2FmZWRkMTJkNGYzZDU1OGEyMThkODZfeG1CbUc3d1ZuYW1KVnJ0M1VMY3ZmTm1EUzJYdkl1RnlfVG9rZW46UFluc2JCVHRkb1JNTEt4cHZPTWozRTN0cDlkXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.1.2. evm

内存:7.7GCPU单核使用率: 71.8%-137.5%，CPU整体使用率: 91.4%

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=MDE0YmM0ODI0ZTE3OWIwMzdkZmM4OWM1YTIwZjY1NDZfdzVVN2hxd2hIYWRKd2RYellHNTNidXgyMnVlUVhxSDVfVG9rZW46UVJHbmJTZm9Db2x2WHF4ekY0a2pPZGZicG1nXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.1.3. exchange

内存: 7.6GCPU单核使用率: 62.5%-123.9%，CPU整体使用率: 83.9%

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=OGE3N2RkMDhhM2YwMWFiYzQ0MmIxODljZDdmMjQ0NWZfVGt3MlBOMjUyQkczRGFXeHRuR2NyenVERHpoSmxHblVfVG9rZW46U0FFZmJwRnNGb0JUUEd4QWV1c2owMjJocHBlXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

### 5.2. 宽带使用情况

#### 5.2.1. bank

上行：平均10.63 MBit/S下行：平均10.63 MBit/S

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=YmY3YjlkMDlhNzc3ZjYyOTY4NjY2ZDMzMTg1OTE0YjlfRU1tQm5NcEV2aWJFZklZVmNyUzVCM2k1d3Zta2tIa2lfVG9rZW46Q2pMQ2J2NFpxb1FTWUV4NlJIUmp2elBYcDJlXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.2.2. evm

上行：平均12.27 MBit/S下行：平均12.27 MBit/S

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=OTQ4MjgzYzc0OGNlN2IyMGU4OTVhZTNkNmQzZWFkMDlfN05sRjJ6WjhPVm5RR3JTMEF0UUJibXFHTHh5bzV6aWJfVG9rZW46RXpNSWJoZG8wb3pSNTd4NU9qeGowM0JucExnXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.2.3. exchange

上行：平均16.28 MBit/S下行：平均16.28 MBit/S

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=Y2NjYTAyNjI1Yzk1NzJlMWM5NjQ5NTQ0YzI0NGFhMzRfR2Q3bGgwYm5uSVl0SDhhdkZ3QWlLUVRsSWYyT1FKZGJfVG9rZW46RW1UbmJyeTBQb05FUzR4Tk9yMmptcnh4cFFiXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

### 5.3. 1分钟测试结果

#### 5.3.1. bank

| 区块数量 | 交易数量  | 平均出块时间（s） | TPS    |
| ---- | ----- | --------- | ------ |
| 47   | 10893 | 1.28      | 181.55 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=NWQwMDRmZTEyZGM1ZjI4YmIxMTVhMThmNjMyODZiOTlfMjhuY0RGa25IV0xablZaRjJuSGdvTmpaSXFQREVrY0lfVG9rZW46TURadmJtelc1b0s3dm14a0dHMGp6N09IcERmXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.3.2. evm

| 区块数量 | 交易数量  | 平均出块时间（s） | TPS    |
| ---- | ----- | --------- | ------ |
| 47   | 11488 | 1.28      | 191.47 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=NWM4NmY2ZTdhMGJkMWU2NmUxOTdhMTYwM2VlMjZiMjFfUjZCTXpxSlhscDNMVzk2bk9FSzlpMGl2VGFCMnZFWEpfVG9rZW46V1RkMmJzNTlMb1lGVzF4dXNNempPdXpycFZkXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.3.3. exchange

| 区块数量 | 交易数量  | 平均出块时间（s） | TPS    |
| ---- | ----- | --------- | ------ |
| 50   | 11387 | 1.2       | 189.78 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=MTMwNTEyM2MyNjBjOWI5NWUwMzllNjQwOTk4MmQyZTNfYlZ5aWw4YTZsNXpib2ZVZm5PMWNiMEFxWXF3MTFIUTdfVG9rZW46RXBXTmJxbXl1bzhzMmR4Vk1DQmpGMDVRcGRjXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

### 5.4. 10分钟测试结果

#### 5.4.1. bank

| 区块数量 | 交易数量   | 平均出块时间（s） | TPS    |
| ---- | ------ | --------- | ------ |
| 401  | 104830 | 1.50      | 174.72 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=NzZjMmQ5ZDE1Y2M1ZTM3OTA4MGU4ZmE0MWY0MTRmOGVfcjdDaWtDbEJvRVBIbVZ3SDdkeFFvbkhuaHpJb0lRS0VfVG9rZW46WGhsN2JCRFpYbzhvT254cGxDVWpFREVWcGFmXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.4.2. evm

| 区块数量 | 交易数量   | 平均出块时间（s） | TPS    |
| ---- | ------ | --------- | ------ |
| 534  | 102781 | 1.12      | 171.30 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=MGU2NWFjMzJmNDI1YTFmYTk2MGJhYTg2MDFhMTI1NzFfcXZwWjdnbURRcjR0SHNzOWdWcElPQ2NUQXdXUnZueDZfVG9rZW46SDFKNWJkUG9sb3FGQlR4dmJybWpDMXpScGpjXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>

#### 5.4.3. exchange

| 区块数量 | 交易数量   | 平均出块时间（s） | TPS    |
| ---- | ------ | --------- | ------ |
| 496  | 103414 | 1.21      | 172.36 |

<figure><img src="https://ojpiha2h9yi6.jp.larksuite.com/space/api/box/stream/download/asynccode/?code=YzdhZTVmNmI0NmFlNmUyOTY5YTZhNDQzMmJjN2YzNDRfbFFOa0g0S2Vjc2hqMW52dFBvekJQOEo0TVBORXZaaTZfVG9rZW46QWR6TGJld0V2b1loOTF4VG5DT2ppc3ZNcDhiXzE3NjI4NjM1OTk6MTc2Mjg2NzE5OV9WNA" alt=""><figcaption></figcaption></figure>
