---
title: TraceTogether简介及其工作原理分析
date: 2020-05-18 09:25:04
tags:
  - TraceTogether
categories:
  - 技普
---

# 前言

最近新加坡疫情比较凶猛，程序员都在家搬砖。在家的好处就是随时可以点点这个看看那个，无痕划水。
几个月前新加坡政府推出了一款用来做 Contact Tracing（接触者跟踪）的手机 APP，并且推荐本地居民安装使用。这个 APP 名字叫[TraceTogether](https://www.tracetogether.gov.sg/)（简称 TT）。当时我虽然也下载安装了，但是以我对政府其他外包项目的认知，感觉这个项目质量不会很高。无非是让 APP 在手机后台一直扫描周围的安装同样 APP 的手机蓝牙的发送的 uuid，然后把这些 id 发送的云端。
前段时间看到新闻说 Apple 和 Google 要联手把 contact tracing 的 API 放到 iOS 和 Android 的系统里，让用户可以知道自己是否和确诊患者有过近距离接触。

出于对自己隐私保护的需要我就去了解了一下这些新 API 是怎么收集数据的，顺便找了一些新加坡 TT 的资料。看了资料后发现 TT 的系统设计的挺不错的。于是就有了这篇文章，聊聊 TT 的实现。

# TraceTogether 系统的组成

从公开的资料来看，TT 系统主要由两个部分组成：用户手机 APP 和云端的 UUID 分发服务。TT 团队将他们的实现方法开源。这个开源的 Contact Tracing 协议叫做[BlueTrace](https://bluetrace.io/)。根据 BlueTrace 协议他们又在 Github 上做了开源实现。这个开源的参考实现项目叫做[OpenTrace](https://github.com/OpenTrace-community)。目前有四个项目：iOS APP(opentrace-ios)， Android APP(opentrace-android)， UUID 云端分发的 Cloud Functions(opentrace-cloud-functions)和手机 APP 蓝牙信号测距的标定(opentrace-calibration)。感兴趣的朋友可以提交一些 PR 试试。
BlueTrace 是系统的协议，相当于 Guideline。下面根据官网提供的[白皮书](https://bluetrace.io/static/bluetrace_whitepaper-938063656596c104632def383eb33b3c.pdf)详细介绍。

# BlueTrace 协议

BlueTrace 协议（下面简称 BT）的实现要求人们在自己手机里面安装它的 APP。这个跨 iOS 和 Android 平台的 APP 当用户下载安装以后会需要蓝牙权限，并且会在后台一直运行。当两台同时运行 BT APP 的手机 A 和 B 相遇（蓝牙可以发现对方，距离约 10 米以内）的时候，他们 APP 就会通过蓝牙协议握手，并且交换它们的临时 ID。此时 A 的 APP 就会记录下：某日某时我和某个（B 的）临时 ID 相遇了。A 并不知道 B 是谁，只是知道有这么一个时间和 ID。而且这个记录只会保留在 A 的手机 APP 的存储空间内，并不会上传到云端服务器上。假如 A 的拥有者确诊了 Covid19，那么卫生部门会要求他把手机 A 上存储信息给上传到卫生部门的服务器上。通过一些数据分析，卫生部门就可以知道 B 手机的拥有者和 A 的接触记录，从而分析出 B 是不是 A 的密切接触者。

## 临时 ID 格式

下面我们看看每个 APP 的临时 ID 格式是怎样的。每个 ID 都是在云端动态生成的，每隔一段时间（推荐的是 15 分钟）会变化一次。ID 的组成如下图所示：
{% asset_img aes-encryption.png 临时ID组成 %}

将用户 ID（User ID，注册时生成），临时 ID 生效时间（Start time）和过期时间（Expiry time）通过 AES-256-GCM 算法加密。加密的密文和加密的初始向量（IV）以及一个认证标签（Auth Tag）组成了最终的 ID。AES 密钥保存在云端（我理解是所有用户使用同一个密钥）。为了在离线情况下继续还能变化 ID，BT 手机 APP 会在联网时把未来 24 小时需要的 ID 全部存储到 APP 里。
{% asset_img tempid-sync.png 下载临时ID %}

下面分析一下 BT 协议这么设计 ID 考虑的一些因素：

**为什么 ID 要加密？**
因为蓝牙握手传输信息都是明文的，想干坏事的人可以写一个简单的蓝牙程序去和周围的手机握手，然后把所有收集到的临时 ID 汇总分析（临时 ID 里面用户 ID）。只要他可以在城市放置足够多的运行这样程序的设备（比如 ESP32，成本只要几美元），他就可以知道某个手机每天去了什么地方，停留了多久。这就和国内的 WIFI 探针玩法一样。

**为什么 ID 要一直变化？**
如果 ID 不变化就会变成一种用户 ID，加密就是失去了意义。当然干坏事的人还可以这么玩：从甲地区读取到 A 的临时 ID，然后再乙地区用 A 的 ID 去和别的机器握手，造成 A 到处乱跑的假象（可能导致 A 被隔离）。使用变化的 ID 就提高了坏人干这个事情的难度，因为他必须要一直跟着 A（为了一直拿到最新的临时 ID）。

## 手机 APP 蓝牙记录密切接触者和测距

分析完 ID 我们再来聊聊手机 APP 蓝牙（BLE）扫描的工作模式。APP 运行时会一直在蓝牙的外设模式（Peripheral）和中心模式（Central）之间切换。中心模式的手机会扫描周围的蓝牙设备，而处于外设模式的手机会广播自己。当处于中心模式的一个手机扫描到另外一个处于外设模式的手机，他们会建立连接并且相互发送一段包含各自临时 ID 的 JSON。
{% asset_img BLE-handshake.png 蓝牙握手流程 %}

外设发给中心的 JSON 格式如下：

```json
{
  // 外设的临时ID
  "id": "Fj5jfbTtDySw8JoVsCmeul0wsoIcJKRPV0 HtEFUlNvNg6C3wyGj8R1utPbw+Iz8tqAdpbxR1 nSvr+ILXPG==",
  // 外设的手机型号
  "mp": "Samsung S8",
  // 外设模式手机用户对应的政府部门，这里是新加坡卫生部
  "o": "SG_MOH",
  // BlueTrace的版本号
  "v": 2
}
```

中心返回给外设的 JSON 格式如下：

```json
{
  // 中心的临时ID
  "id": "Fj5jfbTtDySw8JoVsCmeul0wsoIcJKRPV0 HtEFUlNvNg6C3wyGj8R1utPbw+Iz8tqAdpbxR1 nSvr+ILXPG==",
  //中心的手机型号
  "mc": "iPhone X",
  // 中心设备接收到的RSSI
  "rs": -60,
  // 中心模式手机用户对应的政府部门，这里是新加坡卫生部
  "o": "SG_MOH",
  // BlueTrace的版本号
  "v": 2
}
```

通过中心模式手机测量的 RSSI 可以大概计算出这两个手机的距离。BT 团队也介绍了他们[校准不同品牌手机蓝牙测距的方法](https://github.com/opentrace-community/opentrace-calibration/blob/master/Trial%20Methodologies.md)。这些 JSON 以及收到这些 JSON 的时间戳就是最终被保存在每个 APP 上的数据。当有人确诊，他的 APP 上的数据会被上传到云端服务器用来分析密切接触者。

## 跨国跨部门密切接触者跟踪

我认为这也是 BT 协议比较好的地方。它允许使用它的不同国家的政府和部门之间安全的共享密切接触者的情报。如下图所示，假设 A 是新加坡卫生部，B 是马来西亚卫生部。他们都有自己的一套完整 BT 系统。唯二的不同是他们所使用的 AES 密钥不同，JSON 里面那个 o 不同（一个是 SG_MOH，另一个是 MY_MOH）。
{% asset_img two-health-authorities.png 两个不同国家的用户接触的情况 %}

当新加坡有人确诊，而且他的手机上的数据有马来西亚卫生部 APP 的数据，那么新加坡这边（MOH 的服务器）可以把 JSON 里面的 ID 发给马来西亚（MOH 的服务器）那边，马来那边返回一个伪 ID，如 sha256(userId)。如果最后发现确实有马来那边的密切接触者，新加坡 MOH 可以把那个伪 ID 发给马来那边，相当于通知他们有这么一个人比较危险，可能需要隔离。这样就实现了两国之间密切接触者情报共享，而且保障了各国公民的隐私（公民的个人信息没有被交换）。
{% asset_img cross-border-tracing.png 两个不同国家政府之间匿名信息交换 %}

# 总结

从上面的介绍我们可以看出每一个 BT 手机 APP 用户的数据平时都只保存在自己的手机里，不会上传。只有当一个人确诊的情况下才有必要把信息上传分析。这样在能实现密切接触者追踪的同时最大限度地保护了用户隐私。不一直上传数据还有可能还有考虑到其他的一些因素，比如：
• 政府部门是否可以合法的收集一般居民（健康的人）的信息。
• 上传数据会容易被黑客攻击。所有中心化存储都有这个问题。2018 年发生的 Singhealth 医疗记录泄露事件人们还记忆犹新。与其费力去保护这些数据不如从一开始就不收集这些数据。
