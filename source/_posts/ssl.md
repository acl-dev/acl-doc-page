---
title: 使用SSL中对数据进行加密传输
date: 2020-01-15 13:08:24
tags: SSL编程
categories: SSL编程
---

# 使用SSL中对数据进行加密传输

## 一、概述  

在 Acl 的网络通信模块中，为了支持安全网络传输，引入了第三方 SSL 库，当前支持 OpenSSL, PolarSSL 及其升级版 MbedTLS，Acl 库中通过抽象与封装，大大简化了 SSL 的使用过程（现在开源的 SSL 库使用过程确实过于太复杂），以下是在 Acl 库中使用 SSL 的特点：  
- **动态加载第三方SSL库：** 为了不给非 SSL 用户造成编译上及使用 Acl 库的负担，Acl 库采用动态加载 SSL 动态库方式，这样在编译连接 Acl 库时就不必提供 SSL 库（当然，通过设置编译开关，也允许用户采用静态连接 SSL 库的方式）；  
- **隐藏 SSL 库对外接口：** 在 Acl 的工程中，仅包含了指定版本的 OpenSSL/Polarssl/Mbedtls 头文件（在 acl/include/ 目录下），这些头文件在编译 Acl 的 SSL 模块时会使用到，且不对外暴露，因此使用者需要自行提供对应版本的 SSL 动态二进制库（SSL库的源代码可以去官方 https://tls.mbed.org/ 下载，或者去 https://github.com/acl-dev/third_party 处下载）；
- **模块分离原则：** 在 Acl SSL 模块中，分为全局配置类和 IO 通信类，配置类对象只需在程序启动时进行创建与初始化，且整个进程中按单例方式使用；IO 通信类对象与每一个 TCP 连接所对应的 socket 进行绑定，TCP 连接建立时进行初始化，进行 SSL 握手并接管 IO 过程；
- **支持服务器及客户端模式：** Acl SSL 模块支持服务端及客户端方式，在服务端模块时需要加载数字证书及证书私钥；
- **通信方式：** Acl SSL 模块支持阻塞与非阻塞两种通信方式，阻塞方式还可以用在 Acl 协程通信中；
- **多证书与SNI支持：** 服务端支持加载多个 SSL 证书同时可以根据 SNI 标识加载不同域证书进行 SSL 握手及通信；客户端支持设置 SNI 标识与服务端进行 SSL 握手； 
- **线程安全性：** Acl SSL 模块是线程安全的，虽然官方提供的 Mbedtls 库中增加支持线程安全的编译选项，但其默认情况下却是将此功能关闭的（这真是一个坑人的地方），当你需要打开线程支持功能时还必须得要提供线程锁功能（通过函数回调注册自己的线程锁，好在 Acl 库中有跨平台的线程模块），这应该是 Mbedtls 默认情况下不打开线程支持的原因；
- **应用场景：** Acl SSL 模块已经应用在 Acl HTTP 通信中，从而方便用户编写支持 HTTPS/Websocket 的客户端或服务端程序；同时，Acl SSL 模块也给 Acl Redis 模块提供了安全通信功能；
- **SSL下载：**
  - **下载 MbedTLS：** 当你使用 Mbedtls 时，建议从 https://github.com/acl-dev/third_party/tree/master/mbedtls-2.7.12 下载 Mbedtls 源码编译，此处的 Mbedtls 与官方的主要区别是：
    - 在 config.h 中打开了线程安全的编译选项，同时添加了用于线程安全的互斥锁头文件：threading_alt.h；
    - Mbedtls 库编译后生成了三个库文件：libmbedcrypto/libmbedx509/libmbedtls，而原来 Polarssl 只生成一个库文件，所以为了用户使用方便，修改了 libray/CMakeLists.txt 文件，可以将这三个库文件合并成一个；
    - 增加了 visualc/VC2012（而官方仅提供了 VS2010），这样在 Windows 平台下可以使用 VS 2012 来编译生成 mbedtls 库；
  - **下载 OpenSSL：** 从 OpenSSL 官方或 https://github.com/acl-dev/third_part/ 下载 OpenSSL 1.1.1q 版本。
 
## 二、API 接口说明
为了支持更加通用的 SSL 接口，在 Acl SSL 模块中定义了两个基础类：`sslbase_conf` 和 `sslbase_io`，其中 `ssbase_conf` 类对象可以用做全局单一实例，`ssbase_io` 类对象用来给每一个 TCP socket 对象提供安全 IO 通信功能。

### 2.1、sslbase_conf 类
在 ssbase_conf 类中定义了纯虚方法：`open`，用来创建 SSL IO 通信类对象，在当前所支持的 OpenSSL，Polarssl 和 MbedTSL 中的配置类中（分别为：`acl::openssl_conf`，`acl::polarssl_conf` 和 `acl::mbedtls_conf`）均实现了该方法。下面是 `open` 方法的具体说明：
```c
/**
 * 纯虚方法，创建 SSL IO 对象
 * @param nblock {bool} 是否为非阻塞模式
 * @return {sslbase_io*}
 */
virtual sslbase_io* open(bool nblock) = 0;
```
在客户端或服务端创建 SSL IO 对象（即：sslbase_io 对象）时调用，被用来与 TCP socket 进行绑定。下面是绑定过程：
```c
bool bind_ssl_io(acl::socket_stream& conn, acl::sslbase_conf& ssl_conf) {
	// 创建一个阻塞式 SSL IO 对象
	bool non_block = false;
	acl::sslbase_io* ssl = ssl_conf.open(non_block); 
	
	// 将 SSL IO 对象与 TCP 连接流对象进行绑定，在绑定过程中会进行 SSL 握手，
	// 如果 SSL 握手失败，则返回该 SSL IO 对象，返回 NULL 表示绑定成功。
	if (conn.setup_hook(ssl) == ssl) {
		return false;
	} else {
		return true;
	}
}
```
其中 `acl::sslbase_io` 的父类为 `acl::stream_hook`，在`acl::stream` 流基础类中提供了方法`setup_hook`用来注册外部 IO 过程，其中的参数类型为`stream_hook` ，通过绑定外部 IO 过程，将 SSL IO 过程与 acl 中的流处理 IO 过程进行绑定，从而使 acl 的 IO 流过程具备了 SSL 安全传输能力。 
 
下面的接口用在服务端加载证书及私钥：
```c++
/**
 * 添加一个服务端/客户端自己的证书，可以多次调用本方法加载多个证书
 * @param crt_file {const char*} 证书文件全路径，非空
 * @param key_file {const char*} 密钥文件全路径，非空
 * @param key_pass {const char*} 密钥文件的密码，没有密钥密码可写 NULL
 * @return {bool} 添加证书是否成功
 */
virtual bool add_cert(const char* crt_file, const char* key_file,
        const char* key_pass = NULL);
```

### 2.2、sslbase_io 类
`acl::sslbase_io` 类对象与每一个 TCP 连接对象 `acl::socket_stream` 进行绑定，使 `acl::socket_stream` 具备了进行 SSL 安全通信的能力，在 `acl::sslbase_io`类中声明了纯虚方法`handshake`，这使之成为纯虚类；另外，`acl::sslbase_io` 虽然继承于`acl::stream_hook`类，但并没有实现 `acl::stream_hook` 中规定的四个纯虚方法：`open`，`on_close`，`read`，`send`，这几个虚方法也需要 `acl::sslbase_io`的子类来实现，目前`acl::sslbase_io`的子类有 `acl::openssl_io`，`acl::polarssl_io`及`acl::mbedtls_io` 分别用来支持 `OpenSSL`，`PolarSSL` 及 `MbedTLS`。
下面是这几个纯虚方法的声明：
```c
/**
 * ssl 握手纯虚方法（属于 sslbase_io 类）
 * @return {bool} 返回 true 表示 SSL 握手成功，否则表示失败
 */
virtual bool handshake(void) = 0;
```
下面几个虚方法声明于 `acl::stream_hook` 类中：
```c
/**
 * 读数据接口
 * @param buf {void*} 读缓冲区地址，读到的数据将存放在该缓冲区中
 * @param len {size_t} buf 缓冲区大小
 * @return {int} 读到字节数，当返回值 < 0 时表示出错
 */
virtual int read(void* buf, size_t len) = 0;

/**
 * 发送数据接口
 * @param buf {const void*} 发送缓冲区地址
 * @param len {size_t} buf 缓冲区中数据的长度(必须 > 0) 
 * @return {int} 写入的数据长度，返回值 <０　时表示出错
 */
virtual int send(const void* buf, size_t len) = 0;

/**
 * 在 stream/aio_stream 的 setup_hook 内部将会调用 stream_hook::open
 * 过程，以便于子类对象用来初始化一些数据及会话
 * @param s {ACL_VSTREAM*} 在 setup_hook 内部调用该方法将创建的流对象
 *  作为参数传入
 * @return {bool} 如果子类实例返回 false，则 setup_hook 调用失败且会恢复原样
 */
virtual bool open(ACL_VSTREAM* s) = 0; 

/**
 * 当 stream/aio_stream 流对象关闭前将会回调该函数以便于子类实例做一些善后工作
 * @param alive {bool} 该连接是否依然正常
 * @return {bool}
 */
virtual bool on_close(bool alive) { (void) alive; return true; } 

/**
 * 当 stream/aio_stream 对象需要释放 stream_hook 子类对象时调用此方法
 */
virtual void destroy(void) {}
```
以上几个虚方法均可以在 `acl::openssl_io`，`acl::polarssl_io` 及 `acl::mbedtls_io` 中看到被实现。

## 三、编程示例
### 2.1、服务器模式（使用 OpenSSL）
首先给出一个完整的支持 SSL 的服务端例子，该例子使用了 OpenSSL 做为 SSL 库，如果想切换成 MbedTLS 或 Polarssl 也简单，方法类似（该示例位置：https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/ssl/server）：
```c
#include <assert.h>
#include "lib_acl.h"
#include "acl_cpp/lib_acl.hpp"

class echo_thread : public acl::thread {
public:
        echo_thread(acl::sslbase_conf& ssl_conf, acl::socket_stream* conn)
        : ssl_conf_(ssl_conf), conn_(conn) {}

private:
        acl::sslbase_conf&  ssl_conf_;
        acl::socket_stream* conn_;

        ~echo_thread(void) { delete conn_; }

        // @override
        void* run(void) {
                conn_->set_rw_timeout(60);

                // 给 socket 安装 SSL IO 过程
                if (!setup_ssl()) {
                        return NULL;
                }

                do_echo();

                delete this;
                return NULL;
        }

        bool setup_ssl(void) {
                bool non_block = false;
                acl::sslbase_io* ssl = ssl_conf_.open(non_block);

                // 对于使用 SSL 方式的流对象，需要将 SSL IO 流对象注册至网络
                // 连接流对象中，即用 ssl io 替换 stream 中默认的底层 IO 过程
                if (conn_->setup_hook(ssl) == ssl) {
                        printf("setup ssl IO hook error!\r\n");
                        ssl->destroy();
                        return false;
                }
                return true;
        }

        void do_echo(void) {
                char buf[4096];

                while (true) {
                        int ret = conn_->read(buf, sizeof(buf), false);
                        if (ret == -1) {
                                break;
                        }
                        if (conn_->write(buf, ret) == -1) {
                                break;
                        }
                }
        }
};

static void start_server(const acl::string addr, acl::sslbase_conf& ssl_conf) {
        acl::server_socket ss;
        if (!ss.open(addr)) {
                printf("listen %s error %s\r\n", addr.c_str(), acl::last_serror());
                return;
        }

        while (true) {
                acl::socket_stream* conn = ss.accept();
                if (conn == NULL) {
                        printf("accept error %s\r\n", acl::last_serror());
                        break;
                }
                acl::thread* thr = new echo_thread(ssl_conf, conn);
                thr->set_detachable(true);
                thr->start();
        }
}

static bool ssl_init(const acl::string& ssl_crt, const acl::string& ssl_key,
        acl::mbedtls_conf& ssl_conf) {

        ssl_conf.enable_cache(true);

        // 加载 SSL 证书
        if (!ssl_conf.add_cert(ssl_crt)) {
                printf("add ssl crt=%s error\r\n", ssl_crt.c_str());
                return false;
        }

        // 设置 SSL 证书私钥
        if (!ssl_conf.set_key(ssl_key)) {
                printf("set ssl key=%s error\r\n", ssl_key.c_str());
                return false;
        }

        return true;
}

static void usage(const char* procname) {
        printf("usage: %s -h [help]\r\n"
                " -s listen_addr\r\n"
                " -L ssl_lib_path\r\n"
                " -C crypto_lib_path\r\n"
                " -c ssl_crt\r\n"
                " -k ssl_key\r\n", procname);
}

int main(int argc, char* argv[]) {
        acl::string addr = "0.0.0.0|1443";
#if defined(__APPLE__)
        acl::string crypto_lib = "/usr/local/lib/libcrypto.dylib";
        acl::string ssl_lib = "/usr/local/lib/libssl.dylib";
#elif defined(__linux__)
        acl::string crypto_lib = "/usr/local/lib64/libcrypto.so";
        acl::string ssl_lib = "/usr/local/lib64/libssl.so";
#else
# error "unknown OS type"
#endif
        acl::string ssl_crt = "../ssl_crt.pem", ssl_key = "../ssl_key.pem";

        int ch;
        while ((ch = getopt(argc, argv, "hs:L:C:c:k:")) > 0) {
                switch (ch) {
                case 'h':
                        usage(argv[0]);
                        return 0;
                case 's':
                        addr = optarg;
                        break;
                case 'L':
                        ssl_lib = optarg;
                        break;
                case 'C':
                        crypto_lib = optarg;
                        break;
                case 'c':
                        ssl_crt = optarg;
                        break;
                case 'k':
                        ssl_key = optarg;
                        break;
                default:
                        break;
                }
        }

        acl::log::stdout_open(true);
        
        // 设置 OpenSSL 动态库路径
        // libcrypto, libssl);
        acl::openssl_conf::set_libpath(crypto_lib, ssl_lib);

        // 动态加载 OpenSSL 动态库
        if (!acl::openssl_conf::load()) {
                printf("load %s, %s error\r\n", crypto_lib.c_str(), ssl_lib.c_str());
                return 1;
        }

        // 初始化服务端模式下的全局 SSL 配置对象
        bool server_side = true;

        acl::sslbase_conf* ssl_conf = new acl::openssl_conf(server_side);

        if (!ssl_init(ssl_crt, ssl_key, *ssl_conf)) {
                printf("ssl_init failed\r\n");
                return 1;
        }
        
        start_server(addr, *ssl_conf);

        delete ssl_conf;
        return 0;
}
```
关于该示例有以下几点说明：
- 该服务端例子使用了 OpenSSL 动态库；
- 采用动态加载 OpenSSL 动态库的方式；
- 动态加载时需要设置 OpenSSL 动态库的路径，然后再加载；
- 该例子大体处理流程：
  - 通过 `acl::openssl_conf::set_libpath` 方法设置 OpenSSL 的两个动态库路径，然后调用 `acl::openssl_conf::load` 加载动态库；
  - 在 `ssl_init` 函数中，调用基类 `acl::sslbase_conf` 中的虚方法 `add_cert` 用来加载 SSL 数字证书及证书私钥；
  - 在 `start_server` 函数中，监听本地服务地址，每接收一个 TCP 连接（对应一个 `acl::socket_stream` 对象）便启动一个线程进行 echo 过程；
  - 在客户端处理线程中，调用 `echo_thread::setup_ssl` 方法给该 `acl::socket_stream` TCP 流对象绑定一个 SSL IO 对象，即：先通过调用 `acl::mbedtls_conf::open` 方法创建一个 `acl::mbedtls_io` SSL IO 对象，然后通过 `acl::socket_stream` 的基类中的方法 `set_hook` 将该 SSL IO 对象与 TCP 流对象进行绑定并完成 SSL 握手过程；
  - SSL 握手成功后进入到 `echo_thread::do_echo` 函数中进行简单的 SSL 安全 echo 过程。
 
### 2.2、客户端模式（使用 MbedTLS）
在熟悉了上面的 SSL 服务端编程后，下面给出使用 SSL 进行客户端编程的示例（该示例位置：https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/ssl/client）：
```c
#include <assert.h>
#include "lib_acl.h"
#include "acl_cpp/lib_acl.hpp"

class echo_thread : public acl::thread {
public:
        echo_thread(acl::sslbase_conf& ssl_conf, const char* addr, int count)
        : ssl_conf_(ssl_conf), addr_(addr), count_(count) {}

        ~echo_thread(void) {}

private:
        acl::sslbase_conf&  ssl_conf_;
        acl::string addr_;
        int count_;

private:
        // @override
        void* run(void) {
                acl::socket_stream conn;
                conn.set_rw_timeout(60);
                if (!conn.open(addr_, 10, 10)) {
                        printf("connect %s error %s\r\n",
                                addr_.c_str(), acl::last_serror());
                        return NULL;
                }

                // 给 socket 安装 SSL IO 过程
                if (!setup_ssl(conn)) {
                        return NULL;
                }

                do_echo(conn);

                return NULL;
        }
        bool setup_ssl(acl::socket_stream& conn) {
                bool non_block = false;
                acl::sslbase_io* ssl = ssl_conf_.open(non_block);

                // 对于使用 SSL 方式的流对象，需要将 SSL IO 流对象注册至网络
                // 连接流对象中，即用 ssl io 替换 stream 中默认的底层 IO 过程
                if (conn.setup_hook(ssl) == ssl) {
                        printf("setup ssl IO hook error!\r\n");
                        ssl->destroy();
                        return false;
                }
                printf("ssl setup ok!\r\n");

                return true;
        }

        void do_echo(acl::socket_stream& conn) {
                const char* data = "hello world!\r\n";
                int i;
                for (i = 0; i < count_; i++) {
                        if (conn.write(data, strlen(data)) == -1) {
                                break;
                        }

                        char buf[4096];
                        int ret = conn.read(buf, sizeof(buf) - 1, false);
                        if (ret == -1) {
                                printf("read over, count=%d\r\n", i + 1);
                                break;
                        }
                        buf[ret] = 0;
                        if (i == 0) {
                                printf("read: %s", buf);
                        }
                }
                printf("thread-%lu: count=%d\n", acl::thread::self(), i);
        }
};

static void start_clients(acl::sslbase_conf& ssl_conf, const acl::string addr,
        int cocurrent, int count) {

        std::vector<acl::thread*> threads;
        for (int i = 0; i < cocurrent; i++) {
                acl::thread* thr = new echo_thread(ssl_conf, addr, count);
                threads.push_back(thr);
                thr->start();
        }

        for (std::vector<acl::thread*>::iterator it = threads.begin();
                it != threads.end(); ++it) {
                (*it)->wait(NULL);
                delete *it;
        }
}

static void usage(const char* procname) {
        printf("usage: %s -h [help]\r\n"
                " -s listen_addr\r\n"
                " -L ssl_libs_path\r\n"
                " -c cocurrent\r\n"
                " -n count\r\n", procname);
}

int main(int argc, char* argv[]) {
        acl::string addr = "0.0.0.0|2443";
#if defined(__APPLE__)
        acl::string ssl_lib = "../libmbedtls_all.dylib";
#elif defined(__linux__)
        acl::string ssl_lib = "../libmbedtls_all.so";
#elif defined(_WIN32) || defined(_WIN64)
        acl::string ssl_path = "../mbedtls.dll";

        acl::acl_cpp_init();
#else
# error "unknown OS type"
#endif

        int ch, cocurrent = 10, count = 10;
        while ((ch = getopt(argc, argv, "hs:L:c:n:")) > 0) {
                switch (ch) {
                case 'h':
                        usage(argv[0]);
                        return 0;
                case 's':
                        addr = optarg;
                        break;
                case 'L':
                        ssl_lib = optarg;
                        break;
                case 'c':
                        cocurrent = atoi(optarg);
                        break;
                case 'n':
                        count = atoi(optarg);
                        break;
                default:
                        break;
                }
        }

        acl::log::stdout_open(true);

        // 设置 MbedTLS 动态库路径
        const std::vector<acl::string>& libs = ssl_lib.split2(",; \t");
        if (libs.size() == 1) {
                acl::mbedtls_conf::set_libpath(libs[0]);
        } else if (libs.size() == 3) {
                // libcrypto, libx509, libssl);
                acl::mbedtls_conf::set_libpath(libs[0], libs[1], libs[2]);
        } else {
                printf("invalid ssl_lib=%s\r\n", ssl_lib.c_str());
                return 1;
        }

        // 加载 MbedTLS 动态库
        if (!acl::mbedtls_conf::load()) {
                printf("load %s error\r\n", ssl_lib.c_str());
                return 1;
        }

        // 初始化客户端模式下的全局 SSL 配置对象
        bool server_side = false;

        // SSL 证书校验级别
        acl::mbedtls_verify_t verify_mode = acl::MBEDTLS_VERIFY_NONE;
        acl::mbedtls_conf ssl_conf(server_side, verify_mode);
        start_clients(ssl_conf, addr, cocurrent, count);
        return 0;
}
```
在客户方式下使用 SSL 时的方法与服务端时相似，不同之处是在客户端下使用 SSL 时不必加载证书和设置私钥。

### 2.3、非阻塞模式
在使用 SSL 进行非阻塞编程时，动态库的加载、证书的加载及设置私钥过程与阻塞式 SSL 编程方法相同，不同之处在于创建 SSL IO 对象时需要设置为非阻塞方式，另外在 SSL 握手阶段需要不断检测 SSL 握手是否成功，下面只给出相关不同之处，完整示例可以参考：https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/ssl/aio_server，https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/ssl/aio_client）：

- 调用 `acl::sslbase_conf` 中的虚方法 `open` 时传入的参数为 `true` 表明所创建的 SSL IO 对象为非阻塞方式；
- 在创建非阻塞 IO 对象后，需要调用 `acl::aio_socket_stream` 中的 `read_wait` 方法，以便可以触发 `acl::aio_istream::read_wakeup` 回调，从而在该回调里完成 SSL 握手过程；
- 在非阻塞IO的读回调里需要调用 `acl::sslbase_io` 中的虚方法 `handshake` 尝试进行 SSL 握手并通过 `handshake_ok` 检测握手是否成功。
下面给出在 `read_wakeup` 回调里进行 SSL 握手的过程：
```c
bool read_wakeup()
{
        acl::sslbase_io* hook = (acl::sslbase_io*) client_->get_hook();
        if (hook == NULL) { 
                // 非 SSL 模式，异步读取数据
                //client_->read(__timeout);
                client_->gets(__timeout, false); 
                return true;
        }       

        // 尝试进行 SSL 握手
        if (!hook->handshake()) {
                printf("ssl handshake failed\r\n");
                return false;
        }       

        // 如果 SSL 握手已经成功，则开始按行读数据
        if (hook->handshake_ok()) {
                // 由 reactor 模式转为 proactor 模式，从而取消
                // read_wakeup 回调过程
                client_->disable_read();

                // 异步读取数据，将会回调 read_callback
                //client_->read(__timeout);
                client_->gets(__timeout, false); 
                return true;
        }       

        // SSL 握手还未完成，等待本函数再次被触发
        return true;
}
```
在该代码片断中，如果 SSL 握手一直处于进行中，则 `read_wakeup` 可能会被调用多次，这就意味着 `handshake` 握手过程也会被调用多次，然后再通过 `handshake_ok` 判断握手是否已经成功，如果成果，则通过调用 `gets` 方法切换到 IO 过程（该 IO 过程对应的回调为 `read_callback`），否则进行 SSL 握手过程（继续等待 `read_wakeup` 被回调）。

### 2.4、SNI方式支持使用多个证书
SSL 的早期设计认为一个服务器仅提供一个域名服务，因此只需要加载一个证书即可。但后来因为 HTTP 虚拟主机的产生与发展，只能加载一个 SSL 证书显然是不能满足要求的（服务端与客户端进行 SSL 握手时必须选择合适的域名证书），为此 SSL 协议进行了扩展：在客户端向服务端发起SSL握手阶段便将相应域名信息以明文方式发送至服务端，于时服务器便根据此信息选择相应的 SSL 证书，从而完成了 SSL 过程。在 Acl 的 SSL 模块中也增加了针对 SNI 的支持，从而可以使客户端与服务端进行多域名 SSL 通信。
更多 SSL SNI 内容可以参考：https://abcdxyzk.github.io/blog/2021/06/08/tools-sni/ .

#### 2.4.1、客户端
在 `acl::sslbase_io` 基类中提供了设置 SNI 标识的方法，如下：
```c++
	/**
	 * 设置 SNI HOST 字段
	 * @param host {const char*}
	 */
	void set_sni_host(const char* host);
```
该方法设置了所请求的域名主机标识（对应 `sslbase_io::sni_host_` 成员变量），在 `acl::sslbase_io` 的子类（目前主要是 `acl::openssl_io` 及 `acl::mbedtls_io`）中，当开始 SSL 会话时会首先检测该成员变量（`sni_host_`）是否已经设置，如果已设置，则会自动在 SSL 握手阶段添加 SNI 扩展标识。

#### 2.4.1、服务端
Acl SSL 模块在服务端处理 SSL SNI 的方式是自动的，无需应用特殊处理，只需要使用者添加多个不同域名的证书即可，即多次调用下面方法加载多个域名证书（即调用基类 `acl::sslbase_conf` 中的方法）：
```c++
	/**
	 * 添加一个服务端/客户端自己的证书，可以多次调用本方法加载多个证书
	 * @param crt_file {const char*} 证书文件全路径，非空
	 * @param key_file {const char*} 密钥文件全路径，非空
	 * @param key_pass {const char*} 密钥文件的密码，没有密钥密码可写 NULL
	 * @return {bool} 添加证书是否成功
	 */
	virtual bool add_cert(const char* crt_file, const char* key_file,
		const char* key_pass = NULL);
```
使用者每通过 `sslbase_conf::add_cert` 方法添加一个 SSL 证书，内部人自动解析该证书，并提取证书中记录主机 DNS 信息的扩展数据，并添加主机域名与证书的映射关系；启动运行后，当收到客户端带有 SNI 标识的 SSL 握手请求时，会根据主机域名自动去查找匹配相应的 SSL 证书，从而为该次 SSL 会话选择正确的 SSL 证书。

## 四、Acl 库编译集成第三方 SSL 方式
Acl 库在缺省情况下自动支持 SSL 通信功能并采用动态加载第三方 SSL（OpenSSL/MbedTLS/PolarSSL） 动态库的方式来集成 SSL 功能，查看 Makefile 文件或 VC 工程可以看到在编译 Acl 库时指定了 HAS_OPENSSL_DLL/HAS_MBEDTLS_DLL/HAS_POLARSSL_DLL 编译选项，这样在 openssl_conf.cpp 等与 SSL 相关的功能模块代码中会根据此编译选项动态加载 SSL 动态库；但有时，有的项目希望可以静态链接 SSL 动态库，则在编译 Acl 项目中就需要指定静态连接编译选项，即（下面以集成 OpenSSL 为例）：
```shell
$ cd acl; make OPENSSL_STATIC=yes
```
然后在应用程序自身的 Makefile 中指定 SSL 库，如下 ：
```Makefile
app:
        g++ -o app app.o -lacl_cpp -lprotocol -lacl -lcrypto -lssl -lz -lpthread
```

