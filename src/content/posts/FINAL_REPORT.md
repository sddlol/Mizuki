---
title: BombSquad 1.4.155 bsAccountCycle 逆向工程完整报告
published: 2026-05-30
description: "破解 BombSquad `/bsAccountCycle` 和 `/bsAccountCycleCheap` 协议"
draft: false
author: sddlol
sourceLink: "https://blog.cnmsb.cfd/posts/alicebot/"
---
# BombSquad 1.4.155 bsAccountCycle 逆向工程完整报告

> 作者: CortexaX 🧠  
> 日期: 2026-05-30  
> 目标: 破解 BombSquad `/bsAccountCycle` 和 `/bsAccountCycleCheap` 协议

---

## 📋 前言

jps4 丢给我一个任务：逆向 BombSquad 1.4.155 的账号系统API。

**目标很明确：**
1. 破解 `cs` (checksum) 算法
2. 破解 `d2` 数据加密格式
3. 最终目标：能够模拟任意账号的创建/登录

这看起来是个不可能完成的任务。

---

## 🔍 第一阶段：黑暗中的摸索

### 1.1 最初的线索

从抓包数据中得到：
```
bsAccountCycle:  cs=-4430346, d2=556 bytes
bsAccountCycleCheap: cs=1701867687, d2=89 bytes
ctest: cs=-1727432528, d2=29 bytes
```

所有 `d2` 数据都以 `0x1bee` 开头 —— 这是一个 Magic Number。

### 1.2 cs 算法 —— 无数个死胡同

我尝试了所有能想到的算法：

| 算法 | 结果 |
|------|------|
| CRC32 (zlib) | ✗ |
| Adler32 | ✗ |
| FNV-1a | ✗ |
| MurmurHash3 | ✗ |
| Fletcher32 | ✗ |
| MD5/SHA1 前4字节 | ✗ |
| 简单 XOR | ✗ |
| 各种自创 Hash | ✗ |

**结论（当时）：** cs 必须是某种自定义校验和算法，需要逆向 binary 才能确定。

### 1.3 Frida 动态分析

在 Android 真机上用 Frida hook：
- Hook 了 `PublicEncryptDecrypt` (0x486164)
- Hook 了 `PyPubEncrypt` (0x49b360)
- 一切正常触发，但每次都返回空...

**问题出在哪里？**

---

## 💡 第二阶段：重大突破 —— d2 加密破解

### 2.1 发现 XOR 密钥

在分析 binary 时，在字符串区域发现了关键字符串：

```
"create an account"  (16字节)
```

这是 XOR 加密的密钥！

### 2.2 d2 加密流程验证

```
JSON → zlib.compress → XOR("create an account") → base64 = d2
```

**解密验证（CycleCheap d2）：**
```
原始 d2: G+5oqrVooEF+JbA9nRY4Pn3n36RHZIECQCq2csIUVrr1dWy00BjR+fPmPwn/pSZyNWaUsQFDvZQ4v7d8SmRE/LQ2BpRSe5fPC4QCZEJ/Pib576CSdTNAfFo=

Step 1: base64 decode → 89 bytes
Step 2: XOR with "create an account" (循环16字节密钥)
Step 3: zlib decompress

✅ 解密成功！输出：
{"a":"L-a00994260fd369d4c","c":null,"b":14377,"pp":false,"s":{},"tk":"","ntn":true,"tst":0}
```

Magic `0x1bee` 也完美匹配！

### 2.3 解密后的 JSON 结构

**bsAccountCycleCheap** (`c=null`, 简化版):
```json
{
  "a": "L-a00994260fd369d4c",
  "c": null,
  "b": 14377,
  "pp": false,
  "s": {},
  "tk": "",
  "ntn": true,
  "tst": 0
}
```

**bsAccountCycle** (`c=692867`, 完整版):
```json
{
  "a": "L-a00994260fd369d4c",
  "c": 692867,
  "b": 14377,
  "outstandingTransactions": [[0, {... SET_ACCOUNT_INFO ...}]],
  "pp": false,
  "s": {},
  "tk": "",
  "tst": 0
}
```

---

## 🎯 第三阶段：cs 算法真相大白

### 3.1 转折点 —— 来自 jps4 的抓包文件

jps4 提供了原始抓包数据，其中包含了 curl 命令行可以直接重放的请求！

### 3.2 第一次 curl 尝试 —— 成功！

```bash
curl -X POST 'http://bombsquadgame.com/bsAccountCycle' \
  -H 'User-Agent: BombSquad 1.4.155 TstB (14377)...' \
  --data-urlencode 'cs=-4430346' \
  --data-urlencode 'd2=G+4QMykK+1Fi3Erl/TVV4xGk41rrSVH...'
```

**返回 2992 字节！服务器接受了！**

### 3.3 cs 算法验证 —— 震惊！

用解密后的 JSON 直接计算 CRC32：

```python
json_str = '{"a":"L-a00994260fd369d4c","c":692867,"b":14377,...}'
cs = zlib.crc32(json_str.encode()) & 0xFFFFFFFF
print(cs)  # 4290536950 = 0xffbc65f6 = -4430346 (signed)!
```

**✅ 完全匹配！**

### 3.4 最终结论

| 端点 | cs 算法 |
|------|---------|
| `/bsAccountCycle` | `zlib.crc32(json_bytes) & 0xFFFFFFFF` |
| `/bsAccountCycleCheap` | `zlib.crc32(json_bytes) & 0xFFFFFFFF` |

**两个端点都是标准 CRC32！** 之前无数次的失败是因为——

> **JSON payload 的构建方式不对！**  
> 必须使用精确的 key 顺序和格式化方式：`json.dumps(payload, separators=(',', ':'))`

---

## 🚀 第四阶段：账号生成机制探索

### 4.1 完整的 Python 模拟器

写出了完整的 `bs_simulator.py`，实现了：
- `build_cycle_payload()` - 构建请求 JSON
- `encrypt_d2()` - 加密函数
- `calc_cs()` - CRC32 计算
- `send_request()` - 原始 HTTP 请求
- `decrypt_d2()` - 响应解密

### 4.2 验证结果

```
bsAccountCycle:
  Status: 200
  Response length: 2992 bytes
  Response keys: ['transactionAcks', 't', 'ttp', 'ffap', 'pp', 'ach', 
                 'cmp', 'prch', 'mv', 'mrv', 'mrv2', 'accountID', 
                 'clientID', 'pid', 'delay']

bsAccountCycleCheap:
  Status: 200
  Response length: 172 bytes
  Response keys: ['accountID', 'clientID', 'isNewClientID', 'pid', 'ntn', 'delay']
```

---

## 🎁 第五阶段：最终惊喜 —— UUID 生成任意账号

### 4.1 测试随机 UUID

jps4 问：账号是设备 UUID 生成的，如果用随机 UUID 会怎样？

```python
fake_account_id = "L-a" + str(uuid.uuid4()).replace('-', '')
# 例如: L-a52746a702cfe437bb0f4ffc2da4bd20d
```

**发送请求...**

### 4.2 服务器响应

```
accountID: L-a52746a702cfe437bb0f4ffc2da4bd20d  ← 我们发送的假UUID
clientID: 213486
isNewClientID: true  ← 确认是新账号！
pid: pb-LV4FUxJcUUQRW19CQxAEUFZFQhJfEVUVAgUSDwRGEghdQEQ=  ← 持久化ID
```

**✅ 服务器接受了假 UUID，创建了新账号！**

### 4.3 幂等性验证

用同一个 UUID 发送两次请求：
```
Request 1: clientID=654925, pid=pb-LV4FUxJcUUQ...
Request 2: clientID=412182, pid=pb-LV4FUxJcUUQ...  ← 相同 pid！
```

**同一个 UUID → 同一个账号**（pid 不变，clientID 每次变）

### 4.4 三个随机 UUID 测试

```
UUID 1: L-a0e9d230c80564516be835c9185fb00f9 → pid=pb-LV4FVkVS...
UUID 2: L-a253c8e852fce455aa26d13fbf14eba5c → pid=pb-LV4FVBVY...
UUID 3: L-a62c07c44a61d42feb3e24011b2dea315 → pid=pb-LV4FUBII...
```

**三个随机 UUID → 三个不同账号！**

---

## 📊 最终结论

### BombSquad 本地账号机制

```
┌─────────────────────────────────────────────────────────┐
│                     账号生成流程                          │
├─────────────────────────────────────────────────────────┤
│  1. 客户端生成随机 UUID (32字节, 无横线)                   │
│  2. 格式: "L-a" + 32位UUID = "L-a<device_uuid>"         │
│  3. 服务器接受任意格式正确的 UUID，不做设备验证              │
│  4. 同一 UUID 永远映射到同一 pid (持久化ID)                │
│  5. clientID 每次请求都会变化                             │
└─────────────────────────────────────────────────────────┘
```

### 协议安全性评估

| 方面 | 评估 |
|------|------|
| 账号绑定 | ❌ 无绑定，任何UUID都可创建账号 |
| 设备验证 | ❌ 无验证，不检查设备指纹 |
| 账号上限 | ❌ 可创建任意多个账号 |
| cs 校验 | ✅ CRC32，防止数据篡改 |
| d2 加密 | ⚠️ 弱 XOR 加密，仅防君子 |

**结论：这是一个纯客户端控制的账号系统，服务器信任所有客户端数据。**

---

## 📁 产出文件

| 文件 | 说明 |
|------|------|
| `bs_simulator.py` | 完整的 Python 模拟器，可直接运行 |
| `bsAccountCycle_analysis.md` | 协议分析笔记 |
| `cs_analysis_STATUS.md` | cs 算法分析状态 |

### 使用方法

```python
from bs_simulator import *

# 构建请求
json_str = build_cycle_cheap_payload(account_id="L-a<your_uuid>")
cs = calc_cs(json_str)
d2 = encrypt_d2(json_str)

# 发送请求
response = send_request("/bsAccountCycleCheap", cs, d2)

# 解析响应
data = parse_response(response)
print(data)
```

---

## 🙏 致谢

感谢 jps4 提供的：
- 原始抓包数据
- Android 测试环境
- 持续的反馈和验证

---

## 🎯 经验总结

1. **不要轻易放弃** —— cs 算法测试了20+种方案，最终发现就是标准CRC32
2. **细节决定成败** —— JSON 的 `separators=(',', ':')` 格式化直接影响 CRC32 结果
3. **curl 是好朋友** —— 用 curl 直接重放请求，比写Python更快定位问题
4. **玄学问题的解决** —— 当Python requests返回空但curl成功时，检查HTTP头的细微差别

---

*Report generated by CortexaX 🧠*  
*BombSquad Protocol Analysis - v1.4.155*
