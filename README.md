# sip-call-pitfalls

用 Python 打一通真电话（pjsua2 + linphone.org，机器没有公网 IP）：四处会卡死你的地方。

这里不写安装，不写 API 用法，也不重复官方文档里有的东西。

下面每一条都是「看起来该这么做、其实不行」的地方，以及它真实的症状。四条必须同时成立，缺一个就不通。

---

## 一、不要注册

**踩过的做法**

```python
acfg.regConfig.registrarUri = "sip:sip.linphone.org"
```

**症状**：注册一直超时；或者看着注册成功了，一拨号就没下文。

**正解**

```python
acfg.regConfig.registrarUri = ""
```

**为什么**：免费那个服务不一定需要你先注册。注册走的是另一条通道，和「能不能拨出去」是两件事。把注册关掉、只留 proxy，路反而通。

---

## 二、proxy 必须带 transport

**踩过的做法**

```python
acfg.regConfig.proxy = "sip:sip.linphone.org"
```

**症状**：拨号没反应，日志里也看不出什么。

**正解**：先建一条 TLS transport，再把 proxy 写成带参数的形式。

```python
acfg.regConfig.proxy = "sip:sip.linphone.org;transport=tls"
```

**为什么**：不带 transport 的时候默认按 UDP 走，这条路 UDP 是死的。参数写在字符串里，不写在别处。

---

## 三、SRTP 只能挂在 mediaConfig 这一层

这一条最坑，因为**它不报错**。

**踩过的做法（A）**

```python
epConfig.mediaConfig.srtpUse = pj.PJMEDIA_SRTP_MANDATORY
```

挂到 EpConfig 那一层。设置的时候不报错，顺序看起来也对。

**踩过的做法（B）**

```python
acfg.mediaConfig.transportConfig.srtpUse = pj.PJMEDIA_SRTP_MANDATORY
```

挂到 transportConfig 那一层。

**症状**：调用 `makeCall()` 直接 abort。进程退出码 **134**。没有任何异常信息，没有堆栈，日志里什么都没有。你只会看到程序突然没了。

**正解**

```python
acfg.mediaConfig.srtpUse = pj.PJMEDIA_SRTP_MANDATORY
```

**为什么**：`srtpUse` 是 MediaConfig 自己的字段。挂到别的层，Python 那一层不校验字段归属，设置也「成功」了，一直到 makeCall 那一刻才炸，而且炸得干干净净。

**判断方法**：如果 makeCall 让进程无声无息地死掉，先怀疑这一条的层级，而不是参数值。

---

## 四、STUN 不填 = 通了但没声音

**踩过的做法**

只配了前面三条，STUN 留空。

**症状**：能接通，有回铃，双方都听不见对方。看起来像麦克风权限问题，其实是路的问题。把接通时的 SDP 抓出来看，`c=` 那一行写的是私网地址，比如 `192.168.1.3`。对端按这个地址往回发音频，发到私网里，丢了。

**正解**

```python
acfg.mediaConfig.transportConfig.stunServer = "stun.linphone.org:3478"
```

**为什么**：STUN 的作用就是让 SDP 里写出公网地址，而不是局域网地址。

---

## 怎么判断到底通没通

不看函数返回值，看 SDP。

接通之后把 SDP 打出来，看 `c=` 那一行：

- 私网地址（`10.` / `172.16-31.` / `192.168.`）→ 外面进不来，一定单向或完全没声音
- 公网地址 → 这一层通了

「接通了」和「能听见」是两件事。SDP 分得清，日志分不清。

---

## 四条，缺一不通

```python
acfg.regConfig.registrarUri = ""
acfg.regConfig.proxy = "sip:sip.linphone.org;transport=tls"
acfg.mediaConfig.srtpUse = pj.PJMEDIA_SRTP_MANDATORY
acfg.mediaConfig.transportConfig.stunServer = "stun.linphone.org:3478"
```

四条各自的失败症状：

| 缺哪条 | 症状 |
| --- | --- |
| 一 | 注册超时 / 拨号没下文 |
| 二 | 拨号没反应 |
| 三 | 进程 134 死掉，无任何报错 |
| 四 | 接通了，没有声音 |

症状对不上原因，是因为它们死法完全不一样。

---

## 一句提醒

上面所有的字段位置，是在特定的 pjproject 版本上撞出来的。换版本，字段可能在别处——但「症状 → 层级」的对照关系不会变：**不报错的地方往往最贵**。

---

MIT License
