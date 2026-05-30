# openHiTLS 深度解析：国产高性能 TLS 协议栈从编译到实战

> openHiTLS 是一款面向全场景的开源密码学套件，提供密码算法、TLS/DTLS/TLCP 协议栈、PKI 证书管理等能力。本文从架构设计、编译安装、C API 实战三个维度全面介绍 openHiTLS，帮助开发者快速上手。

---

## 1. openHiTLS 是什么

openHiTLS 是由 openHiTLS 社区（https://openhitls.net）主导开发的开源密码学 C 库，定位类似 OpenSSL，但具有以下差异化优势：

| 对比维度 | OpenSSL | openHiTLS |
|----------|---------|-----------|
| 代码规模 | ~70 万行 | 精简模块化 |
| 国密支持 | 需额外编译/补丁 | 原生支持 SM2/SM3/SM4/TLCP |
| 架构 | 单体 | 高度模块化，按需裁剪 |
| 性能优化 | 通用 | ARM/x86 指令级优化 |
| 后量子算法 | 实验性 | 原生支持 ML-KEM/ML-DSA/SLH-DSA |
| 协议 | TLS 1.0~1.3 | TLS 1.2/1.3 + DTLS 1.2 + TLCP |
| 许可证 | Apache 2.0 | 木兰宽松许可证 v2 (MulanPSL2) |

### 1.1 五大组件

```
openHiTLS
├── BSL (Base Support Layer)    — 基础 C 增强函数、OS 适配、内存管理
├── Crypto                      — 密码算法（对称/非对称/哈希/MAC/KDF/DRBG）
├── TLS                         — 传输层安全协议（TLS/DTLS/TLCP）
├── PKI                         — X.509 证书解析、验证、生成
└── Auth                        — 认证组件（RFC 9578 Publicly Token）
```

BSL 是基础层，其他组件都依赖它。Crypto 可独立使用，TLS 依赖 Crypto + PKI + BSL。

### 1.2 支持的算法

**对称加密：** AES (CBC/GCM/CCM/CTR/XTS)、SM4 (CBC/GCM/CTR/XTS)、ChaCha20-Poly1305

**非对称算法：** RSA、ECDSA、ECDH、DH、DSA、SM2、Ed25519、X25519

**后量子算法：** ML-KEM (Kyber)、ML-DSA (Dilithium)、SLH-DSA (SPHINCS+)

**哈希：** SHA-1、SHA-2 (224/256/384/512)、SHA-3、SM3、MD5、SHAKE

**MAC：** HMAC、CMAC、GMAC、SipHash

**KDF：** HKDF、PBKDF2、SCRYPT、TLS-PRF

**随机数：** DRBG (Hash/HMAC/CTR)

---

## 2. 编译安装

### 2.1 环境要求

| 工具 | 最低版本 | 说明 |
|------|----------|------|
| GCC | 7.3+ | 编译器 |
| CMake | 3.16+ | 构建系统 |
| Python3 | 3.5+ | 配置脚本 |
| Git | 任意 | 拉取代码 |

操作系统：Linux（Ubuntu 20.04/22.04、CentOS 8+、openEuler 等）

### 2.2 获取源码

```bash
# 方式一：含子模块一起拉取（推荐）
git clone --recurse-submodules https://gitcode.com/openhitls/openhitls.git

# 方式二：分步拉取
git clone https://gitcode.com/openhitls/openhitls.git
cd openhitls
git clone https://gitee.com/openeuler/libboundscheck platform/Secure_C
```

openHiTLS 依赖 `libboundscheck`（安全函数库），必须放在 `platform/Secure_C` 目录下。

### 2.3 全量构建（动态库）

```bash
cd openhitls
mkdir -p build && cd build

# 生成配置（全量 + 动态库 + 64位 Linux）
python3 ../configure.py --enable all --lib_type shared --bits 64 --system linux

# 构建
cmake ..
make -j$(nproc)

# 安装到系统
sudo make install

# 刷新动态库缓存
sudo ldconfig
```

默认安装路径为 `/usr/local/`，头文件在 `/usr/local/include/hitls/`，库文件在 `/usr/local/lib/`。

### 2.4 带汇编优化构建

```bash
# x86_64 平台
python3 ../configure.py --enable all --lib_type shared --bits 64 --system linux --asm_type x8664

# ARM v8 平台
python3 ../configure.py --enable all --lib_type shared --bits 64 --system linux --asm_type armv8
```

### 2.5 最小化构建（按需裁剪）

openHiTLS 的核心优势之一是可以按需裁剪。例如只需要 TLS 1.3 + AES-GCM + SHA-256 + ECDHE：

```bash
python3 ../configure.py \
    --enable hitls_bsl hitls_crypto hitls_tls hitls_pki \
    --enable tls13 aes gcm sha2 ecdh ecdsa x25519 hkdf drbg \
    --lib_type static --bits 64 --system linux
```

这种方式生成的库体积远小于全量构建，适合嵌入式场景。

### 2.6 验证安装

```bash
# 检查库文件
ls /usr/local/lib/libhitls_*
# libhitls_bsl.so  libhitls_crypto.so  libhitls_pki.so  libhitls_tls.so

# 检查头文件
ls /usr/local/include/hitls/
# bsl/  crypto/  pki/  tls/
```

### 2.7 命令行工具（可选）

openHiTLS 提供了类似 `openssl` 的命令行工具：

```bash
# 构建命令行工具
python3 ../configure.py --executes hitls --lib_type shared --enable all --asm_type x8664
cmake ..
make -j$(nproc)

# 使用
./hitls help
# 输出: help rand enc pkcs12 rsa x509 list dgst crl genrsa verify passwd pkey genpkey req
```

---

## 3. 架构与核心概念

### 3.1 TLS 模块分层

```
┌─────────────────────────────────────────┐
│            应用层 (Your Code)            │
├─────────────────────────────────────────┤
│  HITLS_Config (配置上下文)               │  ← 一个进程/业务一个
│  HITLS_Ctx    (链路上下文)               │  ← 每条连接一个
├─────────────────────────────────────────┤
│  TLS 协议状态机                          │
│  (握手 / 记录层 / 告警 / 密钥调度)       │
├─────────────────────────────────────────┤
│  BSL_UIO (I/O 抽象层)                   │  ← TCP/UDP/SCTP/自定义
├─────────────────────────────────────────┤
│  Crypto 回调 / PKI 回调                  │  ← 可替换为第三方实现
└─────────────────────────────────────────┘
```

关键设计：
- **Config 与 Ctx 分离**：Config 是模板，Ctx 是实例。一个 Config 可以派生多个 Ctx。
- **I/O 解耦**：TLS 不直接操作 socket，而是通过 BSL_UIO 抽象层。支持阻塞和非阻塞。
- **算法可替换**：通过回调注册机制，可以用第三方密码库替换内置 Crypto。

### 3.2 非阻塞 I/O 模型

openHiTLS 的握手和读写都支持非阻塞模式。当底层 I/O 暂时不可用时，API 返回特定状态码：

| 返回值 | 含义 | 处理方式 |
|--------|------|----------|
| `HITLS_SUCCESS` | 操作完成 | 继续 |
| `HITLS_REC_NORMAL_RECV_BUF_EMPTY` | 接收缓冲区为空 | 等待可读后重试 |
| `HITLS_REC_NORMAL_IO_BUSY` | I/O 忙 | 等待可写后重试 |

典型的非阻塞握手循环：

```c
do {
    ret = HITLS_Connect(ctx);  // 或 HITLS_Accept(ctx)
} while (ret == HITLS_REC_NORMAL_RECV_BUF_EMPTY || ret == HITLS_REC_NORMAL_IO_BUSY);
```

在实际生产中，通常配合 `epoll`/`select` 使用。

---

## 4. C API 实战

### 4.1 初始化

使用 openHiTLS 前需要初始化基础设施：

```c
#include "bsl_sal.h"
#include "bsl_err.h"
#include "crypt_eal_init.h"
#include "hitls_crypt_init.h"
#include "hitls_cert_init.h"

void hitls_global_init(void) {
    // 初始化错误栈
    BSL_ERR_Init();
    
    // 初始化密码算法库
    CRYPT_EAL_Init(CRYPT_EAL_INIT_ALL);
    
    // 注册 TLS 使用的证书回调
    HITLS_CertMethodInit();
    
    // 注册 TLS 使用的密码算法回调
    HITLS_CryptMethodInit();
}
```

### 4.2 TLS 服务端

```c
#include "hitls.h"
#include "hitls_config.h"
#include "hitls_cert.h"
#include "bsl_uio.h"

int tls_server(int listen_fd) {
    // 1. 创建配置
    HITLS_Config *config = HITLS_CFG_NewTLS13Config();
    
    // 2. 加载证书和私钥
    HITLS_CFG_LoadCertFile(config, "server.crt", TLS_PARSE_FORMAT_PEM);
    HITLS_CFG_LoadKeyFile(config, "server.key", TLS_PARSE_FORMAT_PEM);
    
    // 3. 接受 TCP 连接
    int client_fd = accept(listen_fd, NULL, NULL);
    
    // 4. 创建 UIO（I/O 抽象）
    BSL_UIO *uio = BSL_UIO_New(BSL_UIO_TcpMethod());
    BSL_UIO_Ctrl(uio, BSL_UIO_SET_FD, sizeof(client_fd), &client_fd);
    
    // 5. 创建 TLS 链路上下文
    HITLS_Ctx *ctx = HITLS_New(config);
    HITLS_SetUio(ctx, uio);
    
    // 6. TLS 握手
    int32_t ret;
    do {
        ret = HITLS_Accept(ctx);
    } while (ret == HITLS_REC_NORMAL_RECV_BUF_EMPTY || 
             ret == HITLS_REC_NORMAL_IO_BUSY);
    
    if (ret != HITLS_SUCCESS) {
        printf("Handshake failed: 0x%x\n", ret);
        goto cleanup;
    }
    
    // 7. 读写数据
    uint8_t buf[4096];
    uint32_t readLen = 0;
    ret = HITLS_Read(ctx, buf, sizeof(buf), &readLen);
    if (ret == HITLS_SUCCESS) {
        printf("Received %u bytes\n", readLen);
        // Echo back
        uint32_t writeLen = 0;
        HITLS_Write(ctx, buf, readLen, &writeLen);
    }
    
cleanup:
    HITLS_Close(ctx);
    HITLS_Free(ctx);
    BSL_UIO_Free(uio);
    HITLS_CFG_FreeConfig(config);
    close(client_fd);
    return 0;
}
```

### 4.3 TLS 客户端

```c
int tls_client(const char *host, int port) {
    // 1. 创建配置
    HITLS_Config *config = HITLS_CFG_NewTLS13Config();
    
    // 跳过对端证书验证（仅测试用）
    HITLS_CFG_SetVerifyNoneSupport(config, true);
    
    // 2. TCP 连接
    int fd = tcp_connect(host, port);  // 自行实现
    
    // 3. 创建 UIO
    BSL_UIO *uio = BSL_UIO_New(BSL_UIO_TcpMethod());
    BSL_UIO_Ctrl(uio, BSL_UIO_SET_FD, sizeof(fd), &fd);
    
    // 4. 创建 TLS 上下文并握手
    HITLS_Ctx *ctx = HITLS_New(config);
    HITLS_SetUio(ctx, uio);
    
    int32_t ret;
    do {
        ret = HITLS_Connect(ctx);
    } while (ret == HITLS_REC_NORMAL_RECV_BUF_EMPTY || 
             ret == HITLS_REC_NORMAL_IO_BUSY);
    
    if (ret != HITLS_SUCCESS) {
        printf("Connect failed: 0x%x\n", ret);
        goto cleanup;
    }
    
    // 5. 发送数据
    const char *msg = "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n";
    uint32_t writeLen = 0;
    HITLS_Write(ctx, (uint8_t *)msg, strlen(msg), &writeLen);
    
    // 6. 接收响应
    uint8_t buf[4096];
    uint32_t readLen = 0;
    do {
        ret = HITLS_Read(ctx, buf, sizeof(buf), &readLen);
    } while (ret == HITLS_REC_NORMAL_RECV_BUF_EMPTY || 
             ret == HITLS_REC_NORMAL_IO_BUSY);
    
    if (ret == HITLS_SUCCESS) {
        buf[readLen] = '\0';
        printf("Response:\n%s\n", buf);
    }
    
cleanup:
    HITLS_Close(ctx);
    HITLS_Free(ctx);
    BSL_UIO_Free(uio);
    HITLS_CFG_FreeConfig(config);
    close(fd);
    return 0;
}
```

### 4.4 密码套件配置

```c
// TLS 1.3 密码套件
uint16_t ciphers13[] = {
    HITLS_AES_128_GCM_SHA256,       // 0x1301
    HITLS_AES_256_GCM_SHA384,       // 0x1302
    HITLS_CHACHA20_POLY1305_SHA256, // 0x1303
};
HITLS_CFG_SetCipherSuites(config, ciphers13, 3);

// TLS 1.2 密码套件
uint16_t ciphers12[] = {
    HITLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,    // 0xC02F
    HITLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,    // 0xC030
    HITLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,  // 0xC02B
};
HITLS_CFG_SetCipherSuites(config, ciphers12, 3);
```

### 4.5 证书操作 (PKI)

```c
#include "hitls_pki_cert.h"
#include "hitls_pki_x509.h"

void parse_certificate(const char *path) {
    // 解析证书
    HITLS_X509_Cert *cert = NULL;
    int32_t ret = HITLS_X509_CertParseFile(BSL_FORMAT_PEM, path, &cert);
    if (ret != HITLS_PKI_SUCCESS) {
        printf("Parse failed: 0x%x\n", ret);
        return;
    }
    
    // 获取主题
    BSL_Buffer subject = {0};
    HITLS_X509_CertCtrl(cert, HITLS_X509_GET_SUBJECT_DN_STR, &subject, sizeof(BSL_Buffer));
    printf("Subject: %.*s\n", subject.dataLen, subject.data);
    BSL_SAL_Free(subject.data);
    
    // 获取版本
    int32_t version = 0;
    HITLS_X509_CertCtrl(cert, HITLS_X509_GET_VERSION, &version, sizeof(version));
    printf("Version: v%d\n", version + 1);
    
    // 获取公钥
    CRYPT_EAL_PkeyCtx *pubkey = NULL;
    HITLS_X509_CertCtrl(cert, HITLS_X509_GET_PUBKEY, &pubkey, sizeof(pubkey));
    // ... 使用公钥 ...
    CRYPT_EAL_PkeyFreeCtx(pubkey);
    
    // 释放证书
    HITLS_X509_CertFree(cert);
}
```

### 4.6 哈希计算

```c
#include "crypt_eal_md.h"

void sha256_hash(const uint8_t *data, uint32_t len) {
    CRYPT_EAL_MdCTX *ctx = CRYPT_EAL_MdNewCtx(CRYPT_MD_SHA256);
    
    CRYPT_EAL_MdInit(ctx);
    CRYPT_EAL_MdUpdate(ctx, data, len);
    
    uint8_t digest[32];
    uint32_t digestLen = sizeof(digest);
    CRYPT_EAL_MdFinal(ctx, digest, &digestLen);
    
    // 打印哈希值
    for (int i = 0; i < 32; i++) {
        printf("%02x", digest[i]);
    }
    printf("\n");
    
    CRYPT_EAL_MdFreeCtx(ctx);
}
```

### 4.7 TLCP（国密 TLS）

openHiTLS 原生支持 TLCP 1.1（GM/T 0024-2014）：

```c
// 创建 TLCP 配置
HITLS_Config *config = HITLS_CFG_NewTLCPConfig();

// 加载签名证书和加密证书（TLCP 需要双证书）
HITLS_CFG_LoadCertFile(config, "sign_cert.pem", TLS_PARSE_FORMAT_PEM);
HITLS_CFG_LoadKeyFile(config, "sign_key.pem", TLS_PARSE_FORMAT_PEM);
HITLS_CFG_SetTlcpCertificate(config, enc_cert, true, true);  // 加密证书

// 设置国密套件
uint16_t ciphers[] = { 0xE011 };  // ECDHE_SM4_CBC_SM3
HITLS_CFG_SetCipherSuites(config, ciphers, 1);
```

---

## 5. 编译链接你的项目

### 5.1 GCC 命令行

```bash
gcc my_app.c -o my_app \
    -I/usr/local/include/hitls/tls \
    -I/usr/local/include/hitls/crypto \
    -I/usr/local/include/hitls/pki \
    -I/usr/local/include/hitls/bsl \
    -L/usr/local/lib \
    -lhitls_tls -lhitls_pki -lhitls_crypto -lhitls_bsl -lboundscheck \
    -lpthread
```

### 5.2 CMake 集成

```cmake
cmake_minimum_required(VERSION 3.16)
project(my_tls_app)

# 查找 openHiTLS
find_library(HITLS_TLS hitls_tls PATHS /usr/local/lib)
find_library(HITLS_PKI hitls_pki PATHS /usr/local/lib)
find_library(HITLS_CRYPTO hitls_crypto PATHS /usr/local/lib)
find_library(HITLS_BSL hitls_bsl PATHS /usr/local/lib)
find_library(BOUNDSCHECK boundscheck PATHS /usr/local/lib)

add_executable(my_app main.c)
target_include_directories(my_app PRIVATE
    /usr/local/include/hitls/tls
    /usr/local/include/hitls/crypto
    /usr/local/include/hitls/pki
    /usr/local/include/hitls/bsl
)
target_link_libraries(my_app
    ${HITLS_TLS} ${HITLS_PKI} ${HITLS_CRYPTO} ${HITLS_BSL} ${BOUNDSCHECK}
    pthread
)
```

---

## 6. 与 OpenSSL 的 API 对照

对于熟悉 OpenSSL 的开发者，以下对照表帮助快速迁移：

| 功能 | OpenSSL | openHiTLS |
|------|---------|-----------|
| 创建 TLS 配置 | `SSL_CTX_new(TLS_method())` | `HITLS_CFG_NewTLS13Config()` |
| 加载证书 | `SSL_CTX_use_certificate_file()` | `HITLS_CFG_LoadCertFile()` |
| 加载私钥 | `SSL_CTX_use_PrivateKey_file()` | `HITLS_CFG_LoadKeyFile()` |
| 创建连接 | `SSL_new(ctx)` | `HITLS_New(config)` |
| 绑定 fd | `SSL_set_fd(ssl, fd)` | `HITLS_SetUio(ctx, uio)` |
| 客户端握手 | `SSL_connect(ssl)` | `HITLS_Connect(ctx)` |
| 服务端握手 | `SSL_accept(ssl)` | `HITLS_Accept(ctx)` |
| 读数据 | `SSL_read(ssl, buf, len)` | `HITLS_Read(ctx, buf, len, &readLen)` |
| 写数据 | `SSL_write(ssl, buf, len)` | `HITLS_Write(ctx, buf, len, &writeLen)` |
| 关闭 | `SSL_shutdown(ssl)` | `HITLS_Close(ctx)` |
| 释放 | `SSL_free(ssl)` | `HITLS_Free(ctx)` |
| 获取对端证书 | `SSL_get_peer_certificate()` | `HITLS_GetPeerCertificate(ctx)` |
| 设置密码套件 | `SSL_CTX_set_ciphersuites()` | `HITLS_CFG_SetCipherSuites()` |

主要区别：
1. openHiTLS 的 I/O 通过 `BSL_UIO` 抽象，不直接绑定 fd
2. openHiTLS 的读写返回实际长度通过指针参数，而非返回值
3. openHiTLS 需要显式处理非阻塞返回值（`RECV_BUF_EMPTY` / `IO_BUSY`）

---

## 7. 性能优化建议

### 7.1 启用汇编优化

在 x86_64 或 ARMv8 平台上，启用汇编优化可以显著提升密码算法性能：

```bash
# x86_64
python3 ../configure.py --enable all --asm_type x8664 --asm sha2 aes sm4

# ARMv8
python3 ../configure.py --enable all --asm_type armv8 --asm sha2 aes sm4 sm3
```

### 7.2 会话复用

TLS 1.3 支持基于 PSK 的会话恢复，避免完整握手的开销：

```c
// 启用 session ticket
HITLS_CFG_SetSessionTicketSupport(config, true);
```

### 7.3 密码套件选择

优先使用 ECDHE + AES-GCM 组合，兼顾安全性和性能：
- TLS 1.3：`TLS_AES_128_GCM_SHA256` (0x1301)
- TLS 1.2：`ECDHE_RSA_WITH_AES_128_GCM_SHA256` (0xC02F)

---

## 8. 常见问题

### Q: 编译报错 `boundscheck.h: No such file`

安全函数库未正确放置。确认 `platform/Secure_C` 目录存在且已编译：

```bash
cd platform/Secure_C && make -j && cd ../..
```

### Q: 运行时 `libhitls_tls.so: cannot open shared object file`

```bash
# 方法 1：临时生效
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 方法 2：永久生效
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/hitls.conf
sudo ldconfig
```

### Q: 握手返回 `HITLS_REC_NORMAL_RECV_BUF_EMPTY`

这不是错误，是非阻塞 I/O 的正常状态。需要在循环中重试，或配合 epoll 等待 socket 可读后再调用。

### Q: 如何只构建静态库？

```bash
python3 ../configure.py --enable all --lib_type static --bits 64 --system linux
```

### Q: 支持 Windows 吗？

当前官方仅支持 Linux。Windows 支持在规划中。

---

## 9. 社区与贡献

- 官网：https://openhitls.net
- 代码仓库：https://gitcode.com/openhitls/openhitls
- Python 绑定：https://gitcode.com/openHiTLS/pyhitls
- 贡献前需签署 CLA：https://cla.openhitls.net

欢迎通过 Issue 反馈问题，通过 PR 贡献代码。

---

## 10. 总结

openHiTLS 作为国产开源 TLS 协议栈，在以下场景具有独特优势：

1. **国密合规**：原生支持 SM2/SM3/SM4/TLCP，无需额外补丁
2. **嵌入式部署**：模块化裁剪，最小化 ROM/RAM 占用
3. **高性能**：ARM/x86 汇编级优化，适合高并发服务
4. **后量子就绪**：内置 ML-KEM/ML-DSA 等后量子算法
5. **安全合规**：代码精简可审计，适合安全敏感场景

对于 Python 开发者，可以通过 [pyhitls](https://gitcode.com/openHiTLS/pyhitls) 直接使用 openHiTLS 的全部能力，无需编写 C 代码。

---

*本文基于 openHiTLS 2025.09 版本编写，运行环境为 Ubuntu 20.04 x86_64。*
