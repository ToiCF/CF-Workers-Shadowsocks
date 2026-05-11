# CF-Workers-Shadowsocks

`Shadowsocks.js` 是一个基于 Cloudflare Workers 的 Shadowsocks over WebSocket 实验实现：在单文件内完成 **WS 接入 → SS 解密 → 地址解析 → `connect()` 出站 → 回包再加密**。

> 这是研究/学习代码，重点是验证 **旧流密码 + AEAD + Shadowsocks 2022** 能否在 Workers 里统一落地，不承诺生产级吞吐或稳定性。

## 核心能力

当前代码直接覆盖 **18 种对外可用方法**，另含 `plain` 作为 `none` 别名：

- **none**：`none`
- **AEAD**：`aes-128-gcm` / `aes-192-gcm` / `aes-256-gcm` / `chacha20-ietf-poly1305` / `xchacha20-ietf-poly1305`
- **旧流方法**：`chacha20-ietf` / `xchacha20` / `aes-128/192/256-ctr` / `aes-128/192/256-cfb` / `rc4-md5`
- **Shadowsocks 2022**：`2022-blake3-aes-128-gcm` / `2022-blake3-aes-256-gcm` / `2022-blake3-chacha20-poly1305`

> 代码里一共 19 个 key，因为 `plain === none`；对外按 **18 种方法** 统计更准确。

## 节点示例

```text
ss://MjAyMi1ibGFrZTMtYWVzLTEyOC1nY206WVdKalpHVm1aMmhwYW10c2JXNXZjQT09@cf.090227.xyz:2052?plugin=v2ray-plugin%3Bmode%3Dwebsocket%3Bhost%3Dwww.cloudflare.com%3Bpath%3D%2F%3Bmux%3D0#SS
```

对应关系：

- `method = 2022-blake3-aes-128-gcm`
- `psk = YWJjZGVmZ2hpamtsbW5vcA==`
- `plugin = v2ray-plugin; mode=websocket`
- `host = www.cloudflare.com`
- `path = /`
- 端口可用常见 CF HTTP/HTTPS 端口：`80/8080/2052/2082/2086/2095` 或 `443/2053/2083/2087/2096/8443`

**节点里的方法和密码，必须与 Worker 里的 `CFG.method / CFG.pw / CFG.psk` 同步。**

## TLS / noTLS 说明

和通常依赖 TLS 保护传输层的 VLESS/VMess over WS 不同，**Shadowsocks 自身就有加密层**。因此这份实现：

- **支持 TLS**
- **也支持 noTLS**
- **不要求必须走 443 才“安全”**

原因是：**SS 数据在进入 WebSocket 前就已经按所选 method 完成加密**。

但要分清：

- noTLS ≠ 传输特征完全隐藏
- TLS 仍然能额外提供一层传输封装/兼容性
- 真正保证载荷不裸奔的是 **Shadowsocks 自身加密**

## 如何改方法和密码

当前入口配置：

```js
const CFG = { pw: 'test123', psk: 'YWJjZGVmZ2hpamtsbW5vcA==', method: 'aes-256-gcm' };
```

只看三项：

- `method`：选择方法
- `pw`：给 **非 2022** 方法用
- `psk`：给 **2022-blake3** 方法用

### 非 2022 方法

适用：`none/plain`、AEAD、CTR/CFB、`chacha20-*`、`xchacha20`、`rc4-md5`。

```js
const CFG = { pw: 'your-password', psk: 'YWJjZGVmZ2hpamtsbW5vcA==', method: 'aes-256-gcm' };
```

这里 **不是直接拿 `pw` 当最终 key**，而是先走 `EVP_BytesToKey(MD5)` 派生：

- `aes-128-*` → **16 字节 key**
- `aes-192-*` → **24 字节 key**
- `aes-256-*` / `chacha20-*` / `xchacha20-*` → **32 字节 key**
- `rc4-md5` → **16 字节 key**

也就是说，`128 / 192 / 256` 的主要区别是 **最终 key 长度不同**，不是强制要求 `pw` 字符串本身正好写成 16/24/32 字节。

### 2022 方法

适用：

- `2022-blake3-aes-128-gcm`
- `2022-blake3-aes-256-gcm`
- `2022-blake3-chacha20-poly1305`

```js
const CFG = {
  pw: 'test123',
  psk: 'YWJjZGVmZ2hpamtsbW5vcA==',
  method: '2022-blake3-aes-128-gcm'
};
```

这里：

- `pw` 不是主密码
- `psk` 才是 2022 的预共享 key
- 代码会优先把 `psk` 当 **base64** 解码；失败才回退成普通字符串字节

示例里的：

```text
YWJjZGVmZ2hpamtsbW5vcA==
```

解码后是：

```text
abcdefghijklmnop
```

即 **16 字节**，所以它只直接对应：

- `2022-blake3-aes-128-gcm`

如果改成：

- `2022-blake3-aes-256-gcm`
- `2022-blake3-chacha20-poly1305`

那就应改成 **32 字节材料** 的 `psk`，不要继续复用这份 16 字节示例。

## 代码主路径

```text
WebSocket → SS 解密 → 解析目标地址 → TCP 出站
TCP 回包 → SS 加密 → WebSocket
```

代码锚点：

- 配置入口：[`Shadowsocks.js`](./Shadowsocks.js) 第 `2` 行
- 方法表：[`Shadowsocks.js`](./Shadowsocks.js) 第 `3` 行
- BLAKE3 / 2022 subkey：[`Shadowsocks.js`](./Shadowsocks.js) 第 `17-22` 行
- 非 2022 派生：[`Shadowsocks.js`](./Shadowsocks.js) 第 `20`, `34` 行
- AEAD 类：[`Shadowsocks.js`](./Shadowsocks.js) 第 `23` 行
- 流密码类：[`Shadowsocks.js`](./Shadowsocks.js) 第 `24-32` 行
- 地址解析：[`Shadowsocks.js`](./Shadowsocks.js) 第 `46` 行
- WS/TCP 桥接：[`Shadowsocks.js`](./Shadowsocks.js) 第 `47-50` 行

## 与 NekoBox / sing-box / Xray 的对应关系

### NekoBox / sing-box

公开可见的 `NekoBoxForAndroid` Shadowsocks 方法列表，和这份 `Shadowsocks.js` 的 **18 种对外方法名**是直接对齐的；sing-box 的 Shadowsocks inbound/outbound 也是直接按 `method` 字符串工作。

公开源码：

- NekoBox 方法列表：<https://github.com/MatsuriDayo/NekoBoxForAndroid/blob/main/app/src/main/res/values/arrays.xml>
- sing-box option：<https://github.com/SagerNet/sing-box/blob/testing/option/shadowsocks.go>
- sing-box inbound：<https://github.com/SagerNet/sing-box/blob/testing/protocol/shadowsocks/inbound.go>
- sing-box outbound：<https://github.com/SagerNet/sing-box/blob/testing/protocol/shadowsocks/outbound.go>

### Xray

Xray 要分两部分看：

1. **经典 `proxy/shadowsocks`**：公开枚举里明确有 `aes-128-gcm` / `aes-256-gcm` / `chacha20-poly1305` / `xchacha20-poly1305` / `none`
2. **`proxy/shadowsocks_2022`**：单独支持 2022 三种方法

公开源码：

- classic enum：<https://github.com/XTLS/Xray-core/blob/main/proxy/shadowsocks/config.proto>
- classic config：<https://github.com/XTLS/Xray-core/blob/main/proxy/shadowsocks/config.go>
- alias 映射：<https://github.com/XTLS/Xray-core/blob/main/infra/conf/shadowsocks.go>
- 2022 outbound：<https://github.com/XTLS/Xray-core/blob/main/proxy/shadowsocks_2022/outbound.go>
- 2022 test：<https://github.com/XTLS/Xray-core/blob/main/testing/scenarios/shadowsocks_2022_test.go>

### 命名映射

| 本项目 / sing-box / NekoBox | Xray 常见口径 |
| --- | --- |
| `none` | `none` / `plain` |
| `chacha20-ietf-poly1305` | `chacha20-poly1305` / `aead_chacha20_poly1305` / `chacha20-ietf-poly1305` |
| `xchacha20-ietf-poly1305` | `xchacha20-poly1305` / `aead_xchacha20_poly1305` / `xchacha20-ietf-poly1305` |

### 需要明确的一点

这份 `Shadowsocks.js` 的口径是：

- **方法面**：尽量对齐 NekoBox / sing-box 的 SS 方法全集
- **命名面**：兼容 Xray 常见别名
- **实现面**：把旧流密码、AEAD、2022 三条路线统一进一个 Worker 文件

但这 **不等于** 当前公开 `Xray-core` 经典 `proxy/shadowsocks` 枚举已经把本项目这 18 种方法全部等价内建进去。像 `aes-192-gcm`、`chacha20-ietf`、`xchacha20`、`CTR/CFB`、`rc4-md5`，在本项目里已实现、在 NekoBox/sing-box 列表里可见，但不等于公开 Xray 经典枚举也全部同级展开。

## 已知边界

- 主路径是 **TCP over WebSocket**，不是完整 SS UDP 栈
- 旧流方法主要是兼容生态，不代表现代公网优先推荐
- `rc4-md5`、CTR、CFB、旧 `chacha20-*` 都属于兼容性实现
- 2022 路径已单独实现 BLAKE3 / session subkey / 首包头，不是简单套 AEAD 壳

## 文件

| 文件 | 说明 |
| --- | --- |
| [Shadowsocks.js](./Shadowsocks.js) | Worker 主实现：方法表、加解密、地址解析、`connect()` 出站、WebSocket 桥接 |

## 相关链接

- 开源协议：[GPL-3.0](./LICENSE)
- NekoBoxForAndroid：<https://github.com/MatsuriDayo/NekoBoxForAndroid>
- sing-box：<https://github.com/SagerNet/sing-box>
- Xray-core：<https://github.com/XTLS/Xray-core>
- 频道 / 交流群组：<https://t.me/Enkelte_notif>

## Stargazers over time

[![Stargazers over time](https://starchart.cc/ToiCF/CF-Workers-Shadowsocks.svg?variant=adaptive)](https://starchart.cc/ToiCF/CF-Workers-Shadowsocks)
