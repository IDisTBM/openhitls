# PyHiTLS 使用教程：为 openHiTLS 穿上 Python 外衣

> 本文介绍如何在 Ubuntu 环境下编译安装 openHiTLS，并通过其 Python 绑定 pyhitls 快速实现 X.509 证书解析、TLS 1.3 加密通信等功能。适合有一定 Linux 和 Python 基础的开发者。

---

## 1. 项目背景

[openHiTLS](https://gitcode.com/openHiTLS/openhitls) 是一个轻量级、高性能的 TLS 协议栈实现，采用 C 语言编写，支持 TLS 1.2/1.3、国密算法、X.509 PKI 等能力。它的定位类似于 OpenSSL，但代码更精简、模块化更清晰。

**pyhitls** 是 openHiTLS 的官方 Python 绑定，通过 CPython C 扩展直接调用底层 C 库，提供三大模块：

| 模块 | 功能 |
|------|------|
| `pyhitls.crypto` | 对称/非对称加密、哈希、MAC、KDF、随机数 |
| `pyhitls.pki` | X.509 证书解析、字段提取、格式转换 |
| `pyhitls.tls` | TLS 1.2/1.3 客户端和服务端连接 |

本文聚焦 **PKI** 和 **TLS** 两个模块的实战使用。

---

## 2. 环境准备与安装

### 2.1 系统要求

- Ubuntu 20.04+ (本文以 20.04 LTS 为例)
- Python 3.8+
- GCC、CMake 3.16+
- Git

### 2.2 编译安装 openHiTLS

```bash
# 克隆源码
git clone https://gitcode.com/openHiTLS/openhitls.git
cd openhitls

# 配置并编译（CMake 会自动调用 configure.py 生成模块配置）
cmake -B build
cmake --build build -j$(nproc)

# 安装到系统路径
sudo cmake --install build --prefix /usr/local

# 更新动态链接库缓存
sudo ldconfig
```

如果需要定制编译选项（如只启用部分算法），可以在 cmake 之前手动调用 configure.py：

```bash
# 示例：只启用 TLS + PKI + AES + SHA256 + RSA
python3 configure.py --enable tls pki aes sha256 rsa --build_dir build
cmake -B build
cmake --build build -j$(nproc)
```

验证安装：

```bash
ls /usr/local/lib/libhitls_*
# 应看到 libhitls_tls.so, libhitls_pki.so, libhitls_crypto.so, libhitls_bsl.so 等
ls /usr/local/include/hitls/
# 应看到 tls/, pki/, crypto/, bsl/ 等头文件目录
```

### 2.3 安装 pyhitls

```bash
git clone https://gitcode.com/openHiTLS/pyhitls.git
cd pyhitls

# 安装（开发模式，方便调试）
pip3 install -e .

# 或者直接安装
pip3 install .
```

如果遇到链接错误，确保 `LD_LIBRARY_PATH` 包含 openHiTLS 库路径：

```bash
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
```

验证安装：

```python
python3 -c "from pyhitls.pki import Certificate; from pyhitls.tls import TLSContext; print('pyhitls OK')"
```

---

## 3. PKI 模块：X.509 证书操作

### 3.1 生成测试证书

后续示例需要证书和私钥，先用 openssl 生成一套自签名证书：

```bash
# 生成 RSA 2048 私钥 + 自签名证书（有效期 10 年）
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt \
    -days 3650 -nodes -subj "/CN=localhost/O=MyOrg"

# 生成带 SAN 的证书（推荐）
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt \
    -days 3650 -nodes -subj "/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

### 3.2 加载证书

```python
from pyhitls.pki import Certificate

# 从文件加载（默认 PEM 格式）
cert = Certificate.load_from_file('server.crt')

# 从文件加载 DER 格式
cert_der = Certificate.load_from_file('server.der', format='der')

# 从内存字节加载
with open('server.crt', 'rb') as f:
    cert_data = f.read()
cert = Certificate.load_from_bytes(cert_data, format='pem')
```

### 3.3 读取证书字段

```python
cert = Certificate.load_from_file('server.crt')

print(f"主题 (Subject):  {cert.subject}")
print(f"颁发者 (Issuer): {cert.issuer}")
print(f"序列号:          {cert.serial_number}")
print(f"版本:            v{cert.version + 1}")
print(f"生效时间:        {cert.not_before}")
print(f"过期时间:        {cert.not_after}")
```

输出示例：

```
主题 (Subject):  CN=localhost,O=MyOrg
颁发者 (Issuer): CN=localhost,O=MyOrg
序列号:          3A:7B:C1:...
版本:            v3
生效时间:        May 30 08:00:00 2026 GMT
过期时间:        May 28 08:00:00 2036 GMT
```

### 3.4 Subject Alternative Name (SAN)

现代浏览器和客户端已不再信任 CN 字段，而是依赖 SAN 扩展来验证域名：

```python
cert = Certificate.load_from_file('server.crt')

sans = cert.subject_alt_names
if sans:
    for type_, value in sans:
        print(f"  {type_}: {value}")
else:
    print("  该证书没有 SAN 扩展")
```

输出示例：

```
  DNS: localhost
  IP: 127.0.0.1
```

### 3.5 证书指纹

指纹是证书 DER 编码的哈希值，常用于标识和比对证书：

```python
cert = Certificate.load_from_file('server.crt')

# SHA-256 指纹（默认）
print(f"SHA-256: {cert.fingerprint()}")

# SHA-1 指纹
print(f"SHA-1:   {cert.fingerprint('sha1')}")
```

输出示例：

```
SHA-256: FC:EA:49:F8:65:EC:F6:A3:...
SHA-1:   2B:8A:C3:...
```

### 3.6 证书格式转换

```python
cert = Certificate.load_from_file('server.crt')

# PEM → DER
der_bytes = cert.to_der()
with open('server.der', 'wb') as f:
    f.write(der_bytes)

# DER → PEM
pem_bytes = cert.to_pem()

# 直接保存到文件
cert.save_to_file('output.pem', format='pem')
cert.save_to_file('output.der', format='der')
```

### 3.7 证书比较

两个 Certificate 对象可以直接用 `==` 比较（基于 DER 编码）：

```python
cert1 = Certificate.load_from_file('server.crt')
cert2 = Certificate.load_from_bytes(cert1.to_pem())

assert cert1 == cert2  # True：同一证书的不同加载方式
```

### 3.8 资源管理

Certificate 持有 C 层资源，支持 context manager 模式：

```python
with Certificate.load_from_file('server.crt') as cert:
    print(cert.subject)
    print(cert.fingerprint())
# 退出 with 块后 C 资源被释放
```

---

## 4. TLS 模块：加密通信

### 4.1 核心概念

pyhitls 的 TLS 模块设计遵循三层结构：

```
TLSContext          → 配置（协议版本、证书、密码套件、验证模式）
TLSClientConnection → 客户端连接（connect + send/recv）
TLSServerConnection → 服务端连接（accept + send/recv）
```

一个 `TLSContext` 可以被多个连接复用。

### 4.2 最简客户端

```python
import socket
from pyhitls.tls import TLSContext, TLSClientConnection

# 1. 创建 TLS 上下文
ctx = TLSContext(protocol='TLS1.3')
ctx.set_cipher_suites(['TLS_AES_128_GCM_SHA256', 'TLS_AES_256_GCM_SHA384'])

# 2. 建立 TCP 连接
sock = socket.create_connection(('127.0.0.1', 8443), timeout=5)

# 3. TLS 握手 + 数据传输
with TLSClientConnection(ctx) as conn:
    conn.connect(sock)
    
    # 发送 HTTP 请求
    conn.send(b'GET / HTTP/1.1\r\nHost: localhost\r\n\r\n')
    
    # 接收响应
    response = conn.recv(4096)
    print(response.decode())

sock.close()
```

### 4.3 最简服务端

```python
import socket
from pyhitls.tls import TLSContext, TLSServerConnection

# 1. 创建 TLS 上下文并加载证书
ctx = TLSContext(protocol='TLS1.3')
ctx.load_cert_chain('server.crt', 'server.key')

# 2. 监听 TCP
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_sock.bind(('0.0.0.0', 8443))
server_sock.listen(5)
print("TLS server listening on :8443")

while True:
    client_sock, addr = server_sock.accept()
    print(f"New connection from {addr}")
    
    with TLSServerConnection(ctx) as conn:
        try:
            conn.accept(client_sock)
            
            data = conn.recv(4096)
            print(f"Received: {data[:80]}")
            
            conn.send(b'HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nhello')
        except Exception as e:
            print(f"Error: {e}")
    
    client_sock.close()
```

### 4.4 从内存加载证书（适用于 KMS/HSM 场景）

当私钥存储在 KMS、数据库或 HSM 中时，不需要写临时文件：

```python
ctx = TLSContext(protocol='TLS1.3')

# 从数据库/KMS 获取证书和私钥的 PEM 字节
cert_pem = get_cert_from_kms()  # bytes
key_pem = get_key_from_kms()    # bytes

# 直接从内存加载
ctx.load_cert_chain_bytes(cert_pem, key_pem)
```

### 4.5 获取对端证书

握手完成后，客户端可以获取服务端的证书进行自定义验证：

```python
with TLSClientConnection(ctx) as conn:
    conn.connect(sock)
    
    # 获取服务端证书
    peer_cert = conn.get_peer_certificate()
    if peer_cert:
        print(f"Server: {peer_cert.subject}")
        print(f"Fingerprint: {peer_cert.fingerprint()}")
        
        # 自定义验证逻辑（如证书钉扎）
        expected_fp = "FC:EA:49:..."
        if peer_cert.fingerprint() != expected_fp:
            raise Exception("Certificate pinning failed!")
```

### 4.6 TLS 1.2 支持

pyhitls 同时支持 TLS 1.2，使用不同的密码套件名称：

```python
ctx = TLSContext(protocol='TLS1.2')
ctx.set_cipher_suites([
    'ECDHE_RSA_WITH_AES_128_GCM_SHA256',
    'ECDHE_RSA_WITH_AES_256_GCM_SHA384',
    'ECDHE_ECDSA_WITH_AES_128_GCM_SHA256',
])
ctx.load_cert_chain('server.crt', 'server.key')
```

### 4.7 验证模式控制

默认情况下 pyhitls 不验证对端证书（方便开发测试）。生产环境应启用验证：

```python
ctx = TLSContext(protocol='TLS1.3')

# 查看当前验证模式
print(ctx.verify_mode)  # False（默认关闭）

# 启用对端证书验证
ctx.set_verify(True)
```

### 4.8 可用密码套件一览

| 名称 | 协议 | IANA 编号 |
|------|------|-----------|
| `TLS_AES_128_GCM_SHA256` | TLS 1.3 | 0x1301 |
| `TLS_AES_256_GCM_SHA384` | TLS 1.3 | 0x1302 |
| `TLS_CHACHA20_POLY1305_SHA256` | TLS 1.3 | 0x1303 |
| `TLS_AES_128_CCM_SHA256` | TLS 1.3 | 0x1304 |
| `ECDHE_RSA_WITH_AES_128_GCM_SHA256` | TLS 1.2 | 0xC02F |
| `ECDHE_RSA_WITH_AES_256_GCM_SHA384` | TLS 1.2 | 0xC030 |
| `ECDHE_ECDSA_WITH_AES_128_GCM_SHA256` | TLS 1.2 | 0xC02B |
| `ECDHE_ECDSA_WITH_AES_256_GCM_SHA384` | TLS 1.2 | 0xC02C |
| `RSA_WITH_AES_128_GCM_SHA256` | TLS 1.2 | 0x009C |
| `RSA_WITH_AES_256_GCM_SHA384` | TLS 1.2 | 0x009D |

---

## 5. 完整示例：Echo Server

下面是一个完整的 TLS Echo Server + Client 示例，展示双向数据传输：

### 5.1 服务端 (echo_server.py)

```python
#!/usr/bin/env python3
"""TLS Echo Server — 将客户端发来的数据原样返回"""

import socket
import threading
from pyhitls.tls import TLSContext, TLSServerConnection, TLSError


def handle_client(client_sock, addr, ctx):
    """处理单个客户端连接"""
    with TLSServerConnection(ctx) as conn:
        try:
            conn.accept(client_sock)
            print(f"[{addr[0]}:{addr[1]}] TLS 握手成功")

            while True:
                data = conn.recv(4096)
                if not data:
                    break
                print(f"[{addr[0]}:{addr[1]}] 收到 {len(data)} 字节")
                conn.send(data)  # echo back

        except TLSError as e:
            print(f"[{addr[0]}:{addr[1]}] 错误: {e}")

    client_sock.close()
    print(f"[{addr[0]}:{addr[1]}] 连接关闭")


def main():
    ctx = TLSContext(protocol='TLS1.3')
    ctx.load_cert_chain('server.crt', 'server.key')
    ctx.set_cipher_suites(['TLS_AES_256_GCM_SHA384', 'TLS_AES_128_GCM_SHA256'])

    server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_sock.bind(('0.0.0.0', 8443))
    server_sock.listen(5)
    print("Echo Server 启动，监听 :8443 (TLS 1.3)")

    try:
        while True:
            client_sock, addr = server_sock.accept()
            t = threading.Thread(target=handle_client, args=(client_sock, addr, ctx))
            t.daemon = True
            t.start()
    except KeyboardInterrupt:
        print("\n服务器停止")
    finally:
        server_sock.close()


if __name__ == '__main__':
    main()
```

### 5.2 客户端 (echo_client.py)

```python
#!/usr/bin/env python3
"""TLS Echo Client — 发送消息并验证回显"""

import socket
from pyhitls.tls import TLSContext, TLSClientConnection


def main():
    ctx = TLSContext(protocol='TLS1.3')
    ctx.set_cipher_suites(['TLS_AES_256_GCM_SHA384', 'TLS_AES_128_GCM_SHA256'])

    sock = socket.create_connection(('127.0.0.1', 8443), timeout=5)

    with TLSClientConnection(ctx) as conn:
        conn.connect(sock)
        print("TLS 握手成功")

        # 获取并打印服务端证书
        peer = conn.get_peer_certificate()
        if peer:
            print(f"服务端证书: {peer.subject}")
            print(f"指纹: {peer.fingerprint()}")

        # 发送测试消息
        messages = [b'Hello, TLS!', b'pyhitls echo test', b'goodbye']
        for msg in messages:
            conn.send(msg)
            echo = conn.recv(4096)
            status = "OK" if echo == msg else "FAIL"
            print(f"  发送: {msg.decode()} → 回显: {echo.decode()} [{status}]")

    sock.close()
    print("连接关闭")


if __name__ == '__main__':
    main()
```

### 5.3 运行

```bash
# 终端 1：启动服务端
python3 echo_server.py

# 终端 2：运行客户端
python3 echo_client.py
```

输出：

```
# 客户端
TLS 握手成功
服务端证书: CN=localhost,O=MyOrg
指纹: FC:EA:49:F8:65:EC:F6:A3:...
  发送: Hello, TLS! → 回显: Hello, TLS! [OK]
  发送: pyhitls echo test → 回显: pyhitls echo test [OK]
  发送: goodbye → 回显: goodbye [OK]
连接关闭

# 服务端
Echo Server 启动，监听 :8443 (TLS 1.3)
[127.0.0.1:54321] TLS 握手成功
[127.0.0.1:54321] 收到 11 字节
[127.0.0.1:54321] 收到 17 字节
[127.0.0.1:54321] 收到 7 字节
[127.0.0.1:54321] 连接关闭
```

---

## 6. 异常处理

pyhitls 定义了清晰的异常层次：

```
TLSError (基类)
├── TLSConnectionError   # 连接建立/管理错误
├── HandshakeError       # TLS 握手失败
├── TransmissionError    # 数据收发错误
└── ShutdownError        # 连接关闭错误
```

推荐的异常处理模式：

```python
from pyhitls.tls import (
    TLSContext, TLSClientConnection,
    TLSError, HandshakeError, TransmissionError, TLSConnectionError
)

ctx = TLSContext()
sock = socket.create_connection(('example.com', 443), timeout=5)

try:
    with TLSClientConnection(ctx) as conn:
        conn.connect(sock)
        conn.send(b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n')
        data = conn.recv(4096)
except HandshakeError as e:
    print(f"握手失败（证书问题？密码套件不匹配？）: {e}")
except TransmissionError as e:
    print(f"数据传输错误: {e}")
except TLSConnectionError as e:
    print(f"连接错误: {e}")
except TLSError as e:
    print(f"其他 TLS 错误: {e}")
finally:
    sock.close()
```

---

## 7. API 速查表

### 7.1 Certificate

| 方法/属性 | 说明 |
|-----------|------|
| `load_from_file(path, format='pem')` | 从文件加载证书 |
| `load_from_bytes(data, format='pem')` | 从字节加载证书 |
| `.subject` | 主题 DN 字符串 |
| `.issuer` | 颁发者 DN 字符串 |
| `.serial_number` | 序列号（十六进制） |
| `.version` | 版本号（0=v1, 1=v2, 2=v3） |
| `.not_before` / `.not_after` | 有效期 |
| `.subject_alt_names` | SAN 列表 `[(type, value), ...]` |
| `.public_key` | 返回 `PublicKey` 对象 |
| `.fingerprint(algo='sha256')` | 证书指纹 |
| `.to_pem()` / `.to_der()` | 导出为 PEM/DER 字节 |
| `.save_to_file(path, format)` | 保存到文件 |
| `.close()` | 释放 C 资源 |

### 7.2 TLSContext

| 方法/属性 | 说明 |
|-----------|------|
| `TLSContext(protocol='TLS1.3')` | 创建上下文（支持 `'TLS1.2'`、`'TLS1.3'`、`'TLS'`） |
| `.load_cert_chain(certfile, keyfile)` | 从文件加载证书链 |
| `.load_cert_chain_bytes(cert, key)` | 从内存加载证书链 |
| `.set_cipher_suites(names)` | 设置密码套件 |
| `.set_verify(enable)` | 启用/禁用对端验证 |
| `.verify_mode` | 当前验证模式 (bool) |
| `.protocol` | 协议版本字符串 |

### 7.3 TLSClientConnection / TLSServerConnection

| 方法 | 说明 |
|------|------|
| `.connect(sock)` / `.accept(sock)` | TLS 握手 |
| `.send(data) → int` | 发送数据 |
| `.recv(bufsize=4096) → bytes` | 接收数据（对端关闭返回 `b''`） |
| `.get_peer_certificate() → Certificate` | 获取对端证书 |
| `.close()` | 关闭连接 |
| `with ... as conn:` | 支持 context manager |

---

## 8. 常见问题

### Q: 运行时报 `ImportError: libhitls_tls.so: cannot open shared object file`

openHiTLS 库不在系统搜索路径中。解决方法：

```bash
# 方法 1：设置环境变量
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 方法 2：永久生效
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/hitls.conf
sudo ldconfig
```

### Q: 编译 pyhitls 时报找不到头文件

确认 openHiTLS 头文件安装在 `/usr/local/include/hitls/` 下。如果安装路径不同，修改 `setup.py` 中的 `include_dir` 变量。

### Q: TLS 握手超时

确保服务端已加载证书（`load_cert_chain`），且客户端和服务端使用相同的协议版本。pyhitls 当前仅支持阻塞 socket。

### Q: `CertificateStore.verify()` 抛出 `NotImplementedError`

这是已知限制。openHiTLS 底层的证书链验证存在已知问题，pyhitls 暂时禁用了该功能以避免进程崩溃。后续版本会修复。

---

## 9. 与 Python ssl 模块的对比

| 特性 | Python ssl (OpenSSL) | pyhitls (openHiTLS) |
|------|---------------------|---------------------|
| 底层库 | OpenSSL/LibreSSL | openHiTLS |
| 国密支持 | 需要额外编译 | 原生支持 SM2/SM3/SM4 |
| TLS 1.3 | 支持 | 支持 |
| API 风格 | 包装 socket | 独立连接对象 |
| 证书解析 | 有限（需 pyOpenSSL） | 内置完整 X.509 解析 |
| 体积 | 较大 | 轻量 |
| 适用场景 | 通用 | 嵌入式/国密/安全合规 |

---

## 10. 总结

pyhitls 为 Python 开发者提供了一条直接使用 openHiTLS 能力的路径，无需深入 C 编程即可实现：

- X.509 证书的加载、解析、格式转换和指纹计算
- TLS 1.2/1.3 的客户端和服务端加密通信
- 灵活的密码套件配置和验证模式控制

项目仍在积极开发中，欢迎参与贡献：

- 仓库地址：https://gitcode.com/openHiTLS/pyhitls
- openHiTLS 主仓库：https://gitcode.com/openHiTLS/openhitls

---

*本文基于 pyhitls v0.0.1 + openHiTLS (2025.09) 编写，运行环境为 Ubuntu 20.04 + Python 3.8。*
