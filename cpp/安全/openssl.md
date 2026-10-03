OpenSSL 是一个功能强大但学习曲线陡峭的库，我们可以从一个简单的base64编码的代码入手来学习这个库，逐步扩展到 OpenSSL 的核心概念和常见用法。

在深入代码之前了解一下BIO这个概念，全称叫做basic I/O，是openssl提供的io抽象层，类似于c++的iostream，**用于封装和链式组合不同类型的io操作**，比如文件、内存、socket或者base64解编码器等。

```cpp
std::string base64_encode(const unsigned char* input, int length) {
	BIO* bio, * b64;
	BUF_MEM* bufferPtr;
	b64 = BIO_new(BIO_f_base64()); // base64. filter bio
	bio = BIO_new(BIO_s_mem()); // output
	bio = BIO_push(b64, bio); // connect with b64 and bio , push b64
	BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL); // 禁换行，ws握手连续base64字符串
	BIO_write(bio, input, length);
	BIO_flush(bio);
	char* encoded_data;
	long encoded_length = BIO_get_mem_data(bio, &encoded_data);
	std::string result(encoded_data, encoded_length);
	BIO_free_all(bio);
	return result;
}
```

就用这个代码来举例，这就是一个典型的bio链操作，**数据像流过管道一样依次被处理**。开头的BIO_new就是我们的最开始地方，用于创建一个base64的过滤器，BIO_f_base64() 返回一个 "过滤器" BIO 类型，它能将写入的数据进行 Base64 编码[citation:1][citation:9]

**BIO_s_mem返回一个源/目标 BIO类型**，编码后的数据会被写入到这个内存缓冲区中。BIO_push将b64过滤器和bio目标连接在一起，执行之后，数据流向： 写入bio-> b64编码 -> 流入bio(内存目标)。BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL); 设置标志位，告诉base64过滤器不要添加换行符，因为ws的握手等场景要求的是连续的base64字符串。

最后我们写入原始数据进入bio链，经过了过滤器之后会被实时编码，刷新缓冲区，base64编码会以块为单位，BIO_flush确保最后一块数据也被编码和 写入目标，用BIO_get_mem_data来获取内存BIO中数据的指针和长度，这个很高效，最最后就是构造c++字符串并且清理收尾工作，释放整个链，包括b64和bio，无需手动逐个释放。

其中，openssl的内部的内存管理机制，BIO_s_mem创建的是堆内存，内部是由openssl的malloc/calloc分配的，位于进程中的堆段：

```
虚拟内存空间布局（Linux x86_64）：
┌─────────────────┬────── 高地址 (0x7FFFFFFFF000)
│     栈段        │ ← 局部变量、函数调用
├─────────────────┤
│     共享库      │ ← OpenSSL 的 .so 文件映射
├─────────────────┤
│     堆段        │ ← 🔴 encoded_data 指向的内存在这里！
├─────────────────┤
│     BSS段       │ ← 未初始化全局/静态变量
├─────────────────┤
│     数据段      │ ← 初始化全局/静态变量
├─────────────────┤
│     代码段      │ ← 程序指令
└─────────────────┴───── 低地址 (0x400000)
```

## 常见工具函数

使用openssl的时候注意初始化和清理工作：

```cpp
#include <openssl/ssl.h>
#include <openssl/evp.h>
#include <openssl/err.h>

class OpenSSLInitializer {
public:
    OpenSSLInitializer() {
        // 加载所有加密算法
        OpenSSL_add_all_algorithms();
        // 加载所有错误信息
        SSL_load_error_strings();
        // 初始化 SSL 库
        SSL_library_init();
        // 加载配置文件（可选）
        OPENSSL_config(nullptr);
    }
    
    ~OpenSSLInitializer() {
        // 清理全局状态
        EVP_cleanup();
        CRYPTO_cleanup_all_ex_data();
        ERR_free_strings();
    }
};

// 使用 RAII 确保初始化/清理
static OpenSSLInitializer openssl_init;
```

上面这套初始化代码在老项目里非常常见，尤其是兼容 OpenSSL 1.0.x / 1.1.x 的工程。到了较新的 OpenSSL 版本，很多全局初始化已经自动化了，所以你在新代码里经常会看到更简洁的写法。**面试或者读源码的时候，看到这套“显式初始化 + 显式清理”的代码不要慌，它本质上还是 RAII 思维**。

接下来为了让后面的例子不至于全是模板代码，我们先准备几个常见的小工具函数：

### 错误打印

OpenSSL 几乎所有 API 都遵循一个很典型的风格：**返回值告诉你成功失败，详细原因放在错误栈里**。所以平时最常见的辅助函数就是这个：

```cpp
#include <openssl/err.h>
#include <iostream>
#include <stdexcept>

[[noreturn]] void throw_openssl_error(const char* message) {
    std::cerr << message << std::endl;
    ERR_print_errors_fp(stderr);
    throw std::runtime_error(message);
}
```

这个函数的思路很简单：

- `ERR_print_errors_fp(stderr)` 会把当前线程里的 OpenSSL 错误栈按顺序打印出来。
- OpenSSL 的很多函数只返回 `0/1`，没有 C++ 异常，所以我们自己把它封装成异常更顺手。
- 错误栈是线程相关的，**出错之后最好立刻读取**，否则后续操作可能把上下文冲掉。

### 二进制转十六进制

做加密时一定会频繁处理二进制数据，比如 key、iv、nonce、ciphertext、tag。它们通常不是“文本”，直接打印往往会有乱码，所以十六进制输出函数非常常用：

```cpp
#include <iomanip>
#include <sstream>
#include <string>
#include <vector>

std::string to_hex(const std::vector<unsigned char>& data) {
    std::ostringstream oss;
    for (unsigned char byte : data) {
        oss << std::hex << std::setw(2) << std::setfill('0')
            << static_cast<int>(byte);
    }
    return oss.str();
}
```

这里顺便要建立一个习惯：**密钥、IV、密文都应该看作二进制缓冲区，而不是普通字符串**。因为：

- 二进制数据里可能包含 `\0`
- 用 `strlen()` 会被提前截断
- 用 `std::cout << buf` 这种方式也会非常危险

所以后面的例子我们优先使用 `std::vector<unsigned char>` 作为原始字节容器。

### 生成随机字节的辅助函数

```cpp
#include <openssl/rand.h>
#include <vector>

std::vector<unsigned char> random_bytes(int size) {
    std::vector<unsigned char> out(size);
    if (RAND_bytes(out.data(), size) != 1) {
        throw_openssl_error("RAND_bytes failed");
    }
    return out;
}

std::vector<unsigned char> private_random_bytes(int size) {
    std::vector<unsigned char> out(size);
    if (RAND_priv_bytes(out.data(), size) != 1) {
        throw_openssl_error("RAND_priv_bytes failed");
    }
    return out;
}
```

这个工具函数会在后面的**对称加密**里直接拿来生成 key 和 IV。

## 对称加密

OpenSSL 官方更推荐我们通过 `EVP` 这一层来做对称加密。它的好处是**把底层不同算法统一抽象成了一套高层接口**，这样 AES、ChaCha20、GCM、CBC 这些模式都可以走同样的状态机。

简单理解一下，`EVP_CIPHER_CTX` 就像一个“加密任务上下文”：

- 里面记住你当前选的是哪种算法
- 里面记住 key 和 iv
- 里面记住已经处理到第几块
- 里面还记住 padding、tag 等中间状态

所以你会看到 `EncryptInit -> EncryptUpdate -> EncryptFinal` 这种三段式流程，这不是啰嗦，而是因为它天生就是为了支持**流式处理**设计的。

### 理论知识

对称加密中重要的是key和IV这两个要点，key作用是加密和解密的核心秘密，就像保险箱的密码，是必须保密的。IV是初始化向量，用于确保相同的明文每次加密都可以得到不同的密文。不需要保密，但是必须唯一和不可预测，长度等于加密算法的块大小，AES是16字节，加密和解密需要相同的IV。

不使用IV和使用固定IV都是不安全的，不使用IV比使用固定IV更加危险，相同明文块会产生相同的密文块，在电子密码本模式ECB（相同的明文块永远加密成相同的密文本，是最简单的的分组密码工作模式）的情况下：


```
加密过程：
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 明文块 1    │     │ 明文块 2    │     │ 明文块 3    │
│ (16字节)    │     │ (16字节)    │     │ (16字节)    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
   ┌───────┐           ┌───────┐           ┌───────┐
   │  AES  │           │  AES  │           │  AES  │
   │ 加密  │           │ 加密  │           │ 加密  │
   └───────┘           └───────┘           └───────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 密文块 1    │     │ 密文块 2    │     │ 密文块 3    │
└─────────────┘     └─────────────┘     └─────────────┘

关键：每个块独立加密，块之间没有关联！
```

这是最著名的 ECB 问题演示。Linux 的企鹅图标用 ECB 加密后：


```

原始图片:           ECB 加密后:
┌────────┐         ┌────────┐
│ ████   │         │ ██ ██  │
│ ██ ██  │   →     │ ██ ██  │  ← 仍然能看到企鹅轮廓！
│   ██   │         │   ██   │
└────────┘         └────────┘
  企鹅             还能认出是企鹅！
```

虽然说ECB模式不安全，但是理论有个优点，可以并行加密/解密，一个块损坏不会影响其他块，错误不会传播，但是安全不能忽略，它的上位替代是CBC（随机IV）和CTR（随机Nonce）。


### AES-256-CBC 完整例子

先从最经典也最容易理解的 `AES-256-CBC` 开始。它特别适合入门，因为你可以很清楚地看到 `EVP_EncryptInit_ex`、`EVP_EncryptUpdate`、`EVP_EncryptFinal_ex` 这几个核心调用是怎么串起来的。

```cpp
#include <openssl/evp.h>
#include <string>
#include <vector>

std::vector<unsigned char> aes_256_cbc_encrypt(
    const std::vector<unsigned char>& plaintext,
    const std::vector<unsigned char>& key,
    const std::vector<unsigned char>& iv) {

    if (key.size() != 32) {
        throw std::invalid_argument("AES-256 key size must be 32 bytes");
    }
    if (iv.size() != 16) {
        throw std::invalid_argument("AES-CBC IV size must be 16 bytes");
    }

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        throw_openssl_error("EVP_CIPHER_CTX_new failed");
    }

    std::vector<unsigned char> ciphertext(
        plaintext.size() + EVP_MAX_BLOCK_LENGTH);

    int out_len1 = 0;
    int out_len2 = 0;

    if (EVP_EncryptInit_ex(
            ctx, EVP_aes_256_cbc(), nullptr, key.data(), iv.data()) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptInit_ex failed");
    }

    if (EVP_EncryptUpdate(
            ctx,
            ciphertext.data(),
            &out_len1,
            plaintext.data(),
            static_cast<int>(plaintext.size())) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptUpdate failed");
    }

    if (EVP_EncryptFinal_ex(
            ctx,
            ciphertext.data() + out_len1,
            &out_len2) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptFinal_ex failed");
    }

    ciphertext.resize(out_len1 + out_len2);
    EVP_CIPHER_CTX_free(ctx);
    return ciphertext;
}

std::vector<unsigned char> aes_256_cbc_decrypt(
    const std::vector<unsigned char>& ciphertext,
    const std::vector<unsigned char>& key,
    const std::vector<unsigned char>& iv) {

    if (key.size() != 32) {
        throw std::invalid_argument("AES-256 key size must be 32 bytes");
    }
    if (iv.size() != 16) {
        throw std::invalid_argument("AES-CBC IV size must be 16 bytes");
    }

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        throw_openssl_error("EVP_CIPHER_CTX_new failed");
    }

    std::vector<unsigned char> plaintext(
        ciphertext.size() + EVP_MAX_BLOCK_LENGTH);

    int out_len1 = 0;
    int out_len2 = 0;

    if (EVP_DecryptInit_ex(
            ctx, EVP_aes_256_cbc(), nullptr, key.data(), iv.data()) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptInit_ex failed");
    }

    if (EVP_DecryptUpdate(
            ctx,
            plaintext.data(),
            &out_len1,
            ciphertext.data(),
            static_cast<int>(ciphertext.size())) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptUpdate failed");
    }

    if (EVP_DecryptFinal_ex(
            ctx,
            plaintext.data() + out_len1,
            &out_len2) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptFinal_ex failed");
    }

    plaintext.resize(out_len1 + out_len2);
    EVP_CIPHER_CTX_free(ctx);
    return plaintext;
}
```

使用方式如下：

```cpp
#include <iostream>

int main() {
    std::string text = "hello openssl aes-256-cbc";
    std::vector<unsigned char> plaintext(text.begin(), text.end());

    std::vector<unsigned char> key = private_random_bytes(32); // 256 bit
    std::vector<unsigned char> iv = random_bytes(16);          // AES block size

    auto ciphertext = aes_256_cbc_encrypt(plaintext, key, iv);
    auto recovered = aes_256_cbc_decrypt(ciphertext, key, iv);

    std::string recovered_text(recovered.begin(), recovered.end());

    std::cout << "key        : " << to_hex(key) << '\n';
    std::cout << "iv         : " << to_hex(iv) << '\n';
    std::cout << "ciphertext : " << to_hex(ciphertext) << '\n';
    std::cout << "plaintext  : " << recovered_text << '\n';
}
```

这个例子值得慢慢拆一下，因为它几乎就是大多数 `EVP` 对称加密题的原型。

#### 第一步：创建上下文

`EVP_CIPHER_CTX_new()` 会在堆上创建一个 cipher context。你可以把它理解成：

- 这是一次独立的加解密任务对象
- 它保存算法、padding、中间块状态
- 用完必须 `EVP_CIPHER_CTX_free()`

这和前面的 `BIO*` 很像，都是典型的 C 风格资源句柄。

#### 第二步：初始化算法、key、iv

```cpp
EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), nullptr, key.data(), iv.data());
```

这一句做了几件事：

- 指定算法为 `AES-256-CBC`
- 把 key 装进上下文
- 把 iv 装进上下文
- 把内部状态机切到“准备加密”的状态

注意这里的 `EVP_aes_256_cbc()` 返回的不是“直接可用的加密器实例”，而是一个**算法描述对象**。真正的运行状态还是保存在 `ctx` 里面。

#### 第三步：Update 处理大块数据

```cpp
EVP_EncryptUpdate(ctx, out, &out_len, in, in_len);
```

这个 API 设计得非常工程化，它不是只处理一整块内存，而是允许你反复调用：

```cpp
EVP_EncryptUpdate(ctx, out1, &len1, chunk1, chunk1_len);
EVP_EncryptUpdate(ctx, out2, &len2, chunk2, chunk2_len);
EVP_EncryptUpdate(ctx, out3, &len3, chunk3, chunk3_len);
```

这意味着它可以天然地支持：

- 文件分块加密
- 网络流分块加密
- 大数据缓冲区边收边加密

所以它不是“把简单问题复杂化”，而是把简单场景和复杂场景统一到一个接口里。

#### 第四步：Final 收尾

`EVP_EncryptFinal_ex()` 是很多人第一次学 OpenSSL 最容易忽略的步骤。它主要负责：

- 把最后一个不完整分组补齐
- 应用 PKCS#7 padding
- 把最后剩余的密文输出出来

如果你漏掉它，很多情况下密文就不完整了。

解密时的 `EVP_DecryptFinal_ex()` 更重要，因为它还承担了**padding 校验**。如果 key 错了、iv 错了、密文被改了、padding 不合法，这一步就可能失败。

#### 为什么输出缓冲区要开大一点

你会看到我们用了：

```cpp
std::vector<unsigned char> ciphertext(
    plaintext.size() + EVP_MAX_BLOCK_LENGTH);
```

这是因为块加密在带 padding 时，输出可能比输入多一个块。比如 AES block size 是 16 字节，那么哪怕原文已经刚好对齐，也可能多输出 16 字节的 padding 块。

#### CBC 模式的注意点

`AES-256-CBC` 很经典，但也有几个必须记住的点：

- **同一个 key 下，IV 不能重复使用**
- IV 不需要保密，但需要随机且唯一
- CBC 只提供“机密性”，**不提供完整性认证**

最后这一点尤其重要。也就是说，攻击者虽然不一定能直接解密你的内容，但他有机会对密文做位翻转、拼接等攻击。所以现代工程里更推荐直接使用 **AEAD 模式**，比如 `AES-256-GCM` 或 `ChaCha20-Poly1305`。

### AES-256-GCM 例子

GCM 是现代项目里非常常见的模式，因为它不仅能加密，还能顺便完成认证。你可以把它理解成：**一边生成密文，一边生成认证标签 tag**。解密时如果 tag 对不上，OpenSSL 会明确告诉你“这份密文不能信”。

```cpp
#include <array>
#include <vector>

struct GcmEncryptedData {
    std::vector<unsigned char> ciphertext;
    std::array<unsigned char, 16> tag;
};

GcmEncryptedData aes_256_gcm_encrypt(
    const std::vector<unsigned char>& plaintext,
    const std::vector<unsigned char>& key,
    const std::vector<unsigned char>& iv,
    const std::vector<unsigned char>& aad = {}) {

    if (key.size() != 32) {
        throw std::invalid_argument("AES-256-GCM key size must be 32 bytes");
    }

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        throw_openssl_error("EVP_CIPHER_CTX_new failed");
    }

    GcmEncryptedData result;
    result.ciphertext.resize(plaintext.size());

    int len = 0;
    int ciphertext_len = 0;

    if (EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), nullptr, nullptr, nullptr) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptInit_ex failed");
    }

    if (EVP_CIPHER_CTX_ctrl(
            ctx, EVP_CTRL_GCM_SET_IVLEN, static_cast<int>(iv.size()), nullptr) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_CTRL_GCM_SET_IVLEN failed");
    }

    if (EVP_EncryptInit_ex(ctx, nullptr, nullptr, key.data(), iv.data()) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptInit_ex set key/iv failed");
    }

    if (!aad.empty()) {
        if (EVP_EncryptUpdate(
                ctx, nullptr, &len, aad.data(), static_cast<int>(aad.size())) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw_openssl_error("EVP_EncryptUpdate aad failed");
        }
    }

    if (EVP_EncryptUpdate(
            ctx,
            result.ciphertext.data(),
            &len,
            plaintext.data(),
            static_cast<int>(plaintext.size())) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptUpdate plaintext failed");
    }
    ciphertext_len = len;

    if (EVP_EncryptFinal_ex(
            ctx, result.ciphertext.data() + ciphertext_len, &len) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_EncryptFinal_ex failed");
    }
    ciphertext_len += len;
    result.ciphertext.resize(ciphertext_len);

    if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_GET_TAG, 16, result.tag.data()) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_CTRL_GCM_GET_TAG failed");
    }

    EVP_CIPHER_CTX_free(ctx);
    return result;
}

bool aes_256_gcm_decrypt(
    const std::vector<unsigned char>& ciphertext,
    const std::array<unsigned char, 16>& tag,
    const std::vector<unsigned char>& key,
    const std::vector<unsigned char>& iv,
    std::vector<unsigned char>& plaintext,
    const std::vector<unsigned char>& aad = {}) {

    if (key.size() != 32) {
        throw std::invalid_argument("AES-256-GCM key size must be 32 bytes");
    }

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        throw_openssl_error("EVP_CIPHER_CTX_new failed");
    }

    plaintext.resize(ciphertext.size());

    int len = 0;
    int plaintext_len = 0;

    if (EVP_DecryptInit_ex(ctx, EVP_aes_256_gcm(), nullptr, nullptr, nullptr) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptInit_ex failed");
    }

    if (EVP_CIPHER_CTX_ctrl(
            ctx, EVP_CTRL_GCM_SET_IVLEN, static_cast<int>(iv.size()), nullptr) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_CTRL_GCM_SET_IVLEN failed");
    }

    if (EVP_DecryptInit_ex(ctx, nullptr, nullptr, key.data(), iv.data()) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptInit_ex set key/iv failed");
    }

    if (!aad.empty()) {
        if (EVP_DecryptUpdate(
                ctx, nullptr, &len, aad.data(), static_cast<int>(aad.size())) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw_openssl_error("EVP_DecryptUpdate aad failed");
        }
    }

    if (EVP_DecryptUpdate(
            ctx,
            plaintext.data(),
            &len,
            ciphertext.data(),
            static_cast<int>(ciphertext.size())) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_DecryptUpdate ciphertext failed");
    }
    plaintext_len = len;

    if (EVP_CIPHER_CTX_ctrl(
            ctx,
            EVP_CTRL_GCM_SET_TAG,
            static_cast<int>(tag.size()),
            const_cast<unsigned char*>(tag.data())) != 1) {
        EVP_CIPHER_CTX_free(ctx);
        throw_openssl_error("EVP_CTRL_GCM_SET_TAG failed");
    }

    int ret = EVP_DecryptFinal_ex(ctx, plaintext.data() + plaintext_len, &len);
    EVP_CIPHER_CTX_free(ctx);

    if (ret > 0) {
        plaintext_len += len;
        plaintext.resize(plaintext_len);
        return true;
    }

    plaintext.clear();
    return false;
}
```

这段代码和 CBC 最大的区别有三个：

1. `GCM` 可以带 `AAD`，也就是“不加密但要认证”的附加数据，比如协议头、消息类型、版本号。
2. 加密结束后要显式取出 `tag`。
3. 解密时最后的 `EVP_DecryptFinal_ex()` 不只是“收尾”，还承担了**认证校验**。  
   如果 tag 不匹配，它会返回失败，这时你必须把这条消息当作不可信数据丢掉。

这就是为什么现代工程里 GCM 很受欢迎：**机密性 + 完整性** 一次解决，不需要再自己补一层 HMAC。

## 随机数生成

随机数在加密里不是可有可无的辅助品，而是核心资源。key、iv、nonce、session id、挑战值、token，这些东西只要随机性不够，整个系统就可能直接失守。

OpenSSL 官方提供的 `RAND_bytes()` 使用的是 **CSPRNG（cryptographically secure pseudo random number generator）**，也就是密码学安全伪随机数生成器。它和 `std::rand()` 的目标完全不是一个级别：

- `std::rand()` 适合小游戏、简单模拟
- `RAND_bytes()` 才适合密钥材料、nonce、IV、token

### 最常见例子：生成 key、iv、nonce

```cpp
#include <iostream>

int main() {
    auto aes_key = private_random_bytes(32); // 256-bit key
    auto iv = random_bytes(16);              // CBC/GCM 常见 IV
    auto nonce = random_bytes(12);           // GCM 常见 96-bit nonce

    std::cout << "aes_key: " << to_hex(aes_key) << '\n';
    std::cout << "iv     : " << to_hex(iv) << '\n';
    std::cout << "nonce  : " << to_hex(nonce) << '\n';
}
```

这里为什么把 key 和 iv 分成两个函数来演示？

- `RAND_bytes()` 生成普通的加密安全随机字节
- `RAND_priv_bytes()` 语义更强，适合“应该长期保密”的随机值，比如私钥材料、主密钥、会话密钥

可以理解成：**都是安全随机，但 `RAND_priv_bytes()` 更偏“高敏感度私密材料”**。

### 实战里要注意的几个点

#### 1. 一定要检查返回值

这是 OpenSSL 官方文档反复强调的事。主流平台上 OpenSSL 通常会自动从操作系统的熵源播种，但如果熵源失败、PRNG 进入错误状态，它是会明确拒绝生成随机字节的。

所以不要写这种代码：

```cpp
unsigned char key[32];
RAND_bytes(key, sizeof(key)); // 返回值没检查
```

而要写成：

```cpp
unsigned char key[32];
if (RAND_bytes(key, sizeof(key)) != 1) {
    throw_openssl_error("RAND_bytes failed");
}
```

#### 2. 不要再用 `RAND_pseudo_bytes`

这个 API 在较新的 OpenSSL 里已经被废弃了。原因也很直白：**它的语义太容易误导人**，很多人会把“不一定足够安全的伪随机”错当成密钥级别的随机源。

#### 3. `IV` 和 `key` 不是一个概念

很多初学者会有一个误区，觉得“反正都是随机的，那 IV 和 key 差不多”。其实完全不一样：

- `key` 是长期秘密，泄漏就等于加密白做
- `iv/nonce` 通常不要求保密，但要求不重复，尤其是在同一把 key 下

所以生成完之后，**key 要严格保护，IV/nonce 则通常会和密文一起传输**。

## TLS / SSL

终于来到最容易让人“感觉黑盒”的部分了。其实把 TLS/SSL 拆开之后，它的结构并没有想象中神秘：

- 底层还是 TCP
- 上面多套了一层 TLS 记录层
- 握手阶段完成算法协商、证书校验、密钥交换
- 握手完成后，`SSL_read/SSL_write` 看起来像普通读写，但内部已经自动完成了加解密和认证

这里顺便提醒一个容易混淆的点：**虽然今天实际使用的通常是 TLS，但 OpenSSL 的很多 API 名字仍然保留着 `SSL_` 前缀**，这是历史原因，不代表你真的在用过时的 SSLv3 协议。

### 一个最小可运行的 TLS 客户端思路

先给出完整代码，再一点点拆开：

```cpp
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <iostream>
#include <string>

void https_get_example(const std::string& host, const std::string& port) {
    SSL_CTX* ctx = SSL_CTX_new(TLS_client_method());
    if (!ctx) {
        throw_openssl_error("SSL_CTX_new failed");
    }

    // 加载系统默认信任的 CA 证书路径
    if (SSL_CTX_set_default_verify_paths(ctx) != 1) {
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_CTX_set_default_verify_paths failed");
    }

    // 客户端必须主动开启证书校验
    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nullptr);

    BIO* bio = BIO_new_connect((host + ":" + port).c_str());
    if (!bio) {
        SSL_CTX_free(ctx);
        throw_openssl_error("BIO_new_connect failed");
    }

    if (BIO_do_connect(bio) <= 0) {
        BIO_free_all(bio);
        SSL_CTX_free(ctx);
        throw_openssl_error("BIO_do_connect failed");
    }

    SSL* ssl = SSL_new(ctx);
    if (!ssl) {
        BIO_free_all(bio);
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_new failed");
    }

    // SNI：告诉服务器你要访问哪个主机名
    if (SSL_set_tlsext_host_name(ssl, host.c_str()) != 1) {
        BIO_free_all(bio);
        SSL_free(ssl);
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_set_tlsext_host_name failed");
    }

    // 主机名校验：要求证书必须匹配这个域名
    if (SSL_set1_host(ssl, host.c_str()) != 1) {
        BIO_free_all(bio);
        SSL_free(ssl);
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_set1_host failed");
    }

    // 把已经连上的 BIO 交给 SSL 对象接管
    SSL_set_bio(ssl, bio, bio);
    bio = nullptr; // 所有权已经转移给 ssl

    if (SSL_connect(ssl) != 1) {
        SSL_free(ssl);
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_connect failed");
    }

    long verify_result = SSL_get_verify_result(ssl);
    if (verify_result != X509_V_OK) {
        SSL_free(ssl);
        SSL_CTX_free(ctx);
        throw std::runtime_error("certificate verification failed");
    }

    std::cout << "protocol: " << SSL_get_version(ssl) << '\n';
    std::cout << "cipher  : " << SSL_get_cipher(ssl) << '\n';

    std::string request =
        "GET / HTTP/1.1\r\n"
        "Host: " + host + "\r\n"
        "Connection: close\r\n\r\n";

    if (SSL_write(ssl, request.data(), static_cast<int>(request.size())) <= 0) {
        SSL_free(ssl);
        SSL_CTX_free(ctx);
        throw_openssl_error("SSL_write failed");
    }

    char buffer[4096];
    int bytes = 0;
    while ((bytes = SSL_read(ssl, buffer, sizeof(buffer))) > 0) {
        std::cout.write(buffer, bytes);
    }

    SSL_shutdown(ssl);
    SSL_free(ssl);
    SSL_CTX_free(ctx);
}
```

### 第一步：`SSL_CTX` 是“TLS 配置模板”

```cpp
SSL_CTX* ctx = SSL_CTX_new(TLS_client_method());
```

`SSL_CTX` 可以理解成“握手配置模板”或者“TLS 工厂配置”：

- 里面存着协议策略
- 里面存着证书校验策略
- 里面存着 CA 信任链加载位置
- 后面你创建的 `SSL*` 连接对象会继承这些设置

这里选择 `TLS_client_method()` 很重要。它不是“强行指定某个固定 TLS 版本”，而是一个**通用的、可协商的客户端方法**。现代代码通常应该优先用这种版本弹性的入口，而不是手写 `TLSv1_2_client_method()` 这种版本绑定接口。

### 第二步：CA 证书和校验模式

```cpp
SSL_CTX_set_default_verify_paths(ctx);
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nullptr);
```

这两句背后其实分别解决了两个不同的问题：

#### `SSL_CTX_set_default_verify_paths`

它告诉 OpenSSL：**去系统默认信任目录里找 CA 根证书**。

这一步决定的是“我拿什么去验证对方证书链”。

如果这一层没配好，就会出现一种常见问题：

- 对方服务器证书本身没问题
- 但你的客户端找不到对应根证书
- 最后报 `certificate verify failed`

#### `SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nullptr)`

这一步决定的是“我到底要不要认真校验证书”。

很多人第一次用 OpenSSL 时最容易踩的坑就在这。默认情况下如果你不显式打开正确的校验策略，程序可能只是“连上了”，但并不代表它真的安全地验证了对方身份。

所以一句话记住：**TLS 能握手成功，不等于你做对了证书校验。**

### 第三步：SNI 和主机名校验

```cpp
SSL_set_tlsext_host_name(ssl, host.c_str());
SSL_set1_host(ssl, host.c_str());
```

这两句很多时候看起来像重复，实际上它们职责完全不同：

#### `SSL_set_tlsext_host_name`

这是 **SNI（Server Name Indication）**。

它的作用是：在握手阶段就告诉服务器，“我要访问的是这个域名”。这在现代云环境里尤其关键，因为一台服务器或一个负载均衡器后面往往挂着很多域名，服务器需要根据你发来的 host 决定回哪张证书。

#### `SSL_set1_host`

这是 **主机名校验规则**。

它的作用是：即使服务器给了你一张“看起来合法”的证书，OpenSSL 也会继续检查这张证书上的域名是否真的匹配你要访问的 `host`。

比如你访问的是：

```text
api.example.com
```

但对方给你的证书其实是：

```text
evil.example.net
```

那证书链就算来自合法 CA，这张证书对当前连接依然是不对的，必须判失败。

所以这两句其实分别在做：

- `SNI`：告诉服务器“我要哪个站点”
- `SSL_set1_host`：客户端自己校验“你回我的证书是不是这个站点的”

### 第四步：`SSL_connect` 真的在做什么

```cpp
SSL_connect(ssl);
```

这一句就是整个 TLS 握手的入口。它背后会自动完成很多事情：

- 发 `ClientHello`
- 协商 TLS 版本
- 协商 cipher suite
- 收服务端证书链
- 校验证书链
- 做密钥交换
- 推导对称会话密钥
- 建立加密通道

从 OpenSSL 官方文档的角度说，`SSL_connect()` 的语义就是：**在已经准备好的通信通道上，主动发起 TLS/SSL 握手**。

如果底层 BIO 是阻塞式的，那么：

- 成功时它会一直跑到握手完成才返回
- 出错时才提前返回

这就是为什么最小示例里写起来非常像同步 API。

### 第五步：握手之后就进入“安全 socket”模式

握手成功之后，后面的读写就简单很多了：

```cpp
SSL_write(ssl, request.data(), request.size());
SSL_read(ssl, buffer, sizeof(buffer));
```

你可以把它理解成：

- 你写进去的是明文 HTTP 请求
- OpenSSL 在内部自动加密、分帧、认证、发包
- 你读出来的是解密后的明文 HTTP 响应

所以从应用层视角看，它像普通 socket；但从传输层安全视角看，它其实已经在帮你维护整套 TLS 记录协议。

### 为什么还要 `SSL_get_verify_result`

```cpp
long verify_result = SSL_get_verify_result(ssl);
```

这是对证书验证结果的最后一次显式确认。它返回的是 OpenSSL 对对端 X509 证书校验的结果码。

很多时候我们喜欢在样例代码里把这一步也写出来，原因有两个：

- 方便调试，明确知道失败发生在“握手层”还是“证书链校验层”
- 读代码的人更容易意识到：**TLS 安全性并不只靠 `SSL_connect()` 一个返回值**

### 如果你要写服务端

客户端是 `TLS_client_method()` + `SSL_connect()`，那服务端就是镜像关系：

- `TLS_server_method()`
- `SSL_CTX_use_certificate_file()`
- `SSL_CTX_use_PrivateKey_file()`
- `SSL_accept()`

也就是说服务端除了握手本身，还必须额外准备：

- 自己的证书
- 自己的私钥
- 可选的 client certificate 验证策略

可以把它理解成：**客户端主要负责“验别人”，服务端主要负责“证明自己”**。

## 这一章最值得背下来的几个点

如果把这一整章浓缩成几个必须形成肌肉记忆的结论，大概就是这些：

1. **对称加密优先走 `EVP` 高层接口**，不要一上来就找底层算法实现。
2. `EncryptInit -> Update -> Final` 是 OpenSSL 的标准状态机，不是样板噪音。
3. `CBC` 只负责保密，不负责认证；现代项目更推荐 `GCM`。
4. `RAND_bytes()` / `RAND_priv_bytes()` 才是密钥级别随机源，别用 `std::rand()`。
5. `TLS_client_method()` 是现代客户端更常见的入口，避免绑定死某个具体协议版本。
6. TLS 连接成功不等于真的安全，**证书链、主机名、CA 信任路径都必须配对**。
7. OpenSSL 的大部分错误细节都在错误栈里，返回值失败后第一时间看 `ERR_print_errors_fp`。

当你把这些点吃透之后，OpenSSL 就不再是“一个充满黑魔法的 C 库”，而更像是：

- 一套 BIO 风格的 IO 抽象
- 一套 EVP 风格的密码学抽象
- 一套 SSL/TLS 风格的握手与安全通道抽象

真正难的地方不在函数名，而在于你要时刻知道：**你当前在处理的是文本、二进制、密钥材料、还是证书与信任链**。一旦这个抽象层次分清楚了，OpenSSL 的代码就会顺很多。
