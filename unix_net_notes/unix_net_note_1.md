# 第一部分 简介和TCP/IP

## 第一章 简介

笔记跳过繁琐的概念背景描述，默认有一定的网络和编程基础。

TCP/IP中两个被广泛使用协议的典型案例：

#### NTP 客户端的案例（UDP）

书中原文是使用tcp获取时间，我这里为了方便直接使用公网上的NTP server，只能使用udp连接，实验代码如下（代码主体由claude.ai生成，仅作编译调试）：

**ntp_client_udp.c**

~~~c
/* Create by Claude.ai 2025-11-21 
 * A NTP client program using UDP.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netdb.h>
#include <sys/types.h>

#define NTP_TIMESTAMP_DELTA 2208988800LL
#define NTP_PACKET_SIZE 48

typedef struct {
    uint8_t li_vn_mode;
    uint8_t stratum;
    uint8_t poll;
    uint8_t precision;
    uint32_t root_delay;
    uint32_t root_dispersion;
    uint32_t ref_id;
    uint32_t ref_ts_sec;
    uint32_t ref_ts_frac;
    uint32_t orig_ts_sec;
    uint32_t orig_ts_frac;
    uint32_t recv_ts_sec;
    uint32_t recv_ts_frac;
    uint32_t tx_ts_sec;
    uint32_t tx_ts_frac;
} ntp_packet;

time_t get_ntp_time(const char *host, int port) {
    int sock;
    struct sockaddr_in addr;
    struct hostent *he;
    ntp_packet pkt;
    unsigned char buf[NTP_PACKET_SIZE];
    time_t ntp_time;
    struct timeval tv;

    // Resolve hostname
    he = gethostbyname(host);
    if (!he) {
        perror("gethostbyname");
        return -1;
    }

    printf("Resolved %s to %s\n", host, inet_ntoa(*(struct in_addr *)he->h_addr_list[0]));

    // Create UDP socket
    sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("socket");
        return -1;
    }

    // Set receive timeout
    tv.tv_sec = 5;
    tv.tv_usec = 0;
    setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (const char *)&tv, sizeof(tv));

    // Set up address structure
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    memcpy(&addr.sin_addr, he->h_addr_list[0], he->h_length);

    // Initialize NTP packet
    memset(&pkt, 0, sizeof(pkt));
    pkt.li_vn_mode = 0x1B;  // LI=0, VN=3, Mode=3 (Client)
    pkt.poll = 4;
    pkt.precision = 0xEC;

    printf("Sending NTP request...\n");

    // Send NTP request via UDP
    if (sendto(sock, (unsigned char *)&pkt, NTP_PACKET_SIZE, 0, 
               (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("sendto");
        close(sock);
        return -1;
    }

    printf("Waiting for response...\n");

    // Receive NTP response
    int n = recvfrom(sock, buf, NTP_PACKET_SIZE, 0, NULL, NULL);
    if (n < NTP_PACKET_SIZE) {
        perror("recvfrom");
        close(sock);
        return -1;
    }

    close(sock);

    // Extract transmit timestamp (seconds since 1900)
    uint32_t tx_sec = (buf[40] << 24) | (buf[41] << 16) | 
                      (buf[42] << 8) | buf[43];

    // Convert from NTP epoch (1900) to Unix epoch (1970)
    ntp_time = tx_sec - NTP_TIMESTAMP_DELTA;

    return ntp_time;
}

int main() {
    const char *ntp_server = "pool.ntp.org";
    int port = 123;
    time_t ntp_time;
    struct tm *tm_info;
    char time_str[80];

    printf("Connecting to NTP server: %s:%d\n", ntp_server, port);

    ntp_time = get_ntp_time(ntp_server, port);

    if (ntp_time < 0) {
        fprintf(stderr, "Failed to get NTP time\n");
        return 1;
    }
    // Convert to readable format
    tm_info = localtime(&ntp_time);
    
    // Format options:
    strftime(time_str, sizeof(time_str), "%Y-%m-%d %H:%M:%S", tm_info);
    printf("Unix timestamp: %ld\n", ntp_time);
    printf("Format 1 (Date and Time): %s\n", time_str);
    
    return 0;
}
~~~

~~~(空)
$ make ntp_client_udp 
cc -o ntp_client_udp ntp_client_udp.c

$ ./ntp_client_udp 
Connecting to NTP server: pool.ntp.org:123
Resolved pool.ntp.org to 139.199.215.251
Sending NTP request...
Waiting for response...
Unix timestamp: 1763713904
Format 1 (Date and Time): 2025-11-21 16:31:44
~~~

#### Web 客户端的案例（TCP）

为补充TCP使用的案例，设计简易的Web客户端，实验代码如下（代码主体由claude.ai生成，仅作编译调试）：

**web_client_tcp.c**

~~~c
/* Create by Claude.ai 2025-11-21 
 * A web client (TCP) that supports both HTTP and HTTPS.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netdb.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

#define BUFFER_SIZE 4096

int connect_http(const char *hostname, int port, const char *path) {
    int sock;
    struct sockaddr_in server_addr;
    struct hostent *he;
    char buffer[BUFFER_SIZE];

    printf("Using HTTP (plain text)\n\n");

    // Resolve hostname
    he = gethostbyname(hostname);
    if (!he) {
        perror("gethostbyname failed");
        return 1;
    }

    printf("Resolved IP: %s\n\n", inet_ntoa(*(struct in_addr *)he->h_addr_list[0]));

    // Create TCP socket
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket creation failed");
        return 1;
    }

    // Set up server address
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    memcpy(&server_addr.sin_addr, he->h_addr_list[0], he->h_length);

    // Connect to server
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Connection failed");
        close(sock);
        return 1;
    }

    printf("Connected successfully!\n\n");

    // Build HTTP request
    char http_request[BUFFER_SIZE];
    snprintf(http_request, sizeof(http_request),
             "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n\r\n",
             path, hostname);

    printf("Sending HTTP request...\n");
    printf("------- REQUEST -------\n%s", http_request);
    printf("------------------------\n\n");

    // Send HTTP request
    if (send(sock, http_request, strlen(http_request), 0) < 0) {
        perror("send failed");
        close(sock);
        return 1;
    }

    printf("Receiving response...\n");
    printf("------- RESPONSE -------\n");

    // Receive response
    int bytes_received;
    while ((bytes_received = recv(sock, buffer, sizeof(buffer) - 1, 0)) > 0) {
        buffer[bytes_received] = '\0';
        printf("%s", buffer);
    }

    printf("\n------------------------\n");

    close(sock);
    return 0;
}

int connect_https(const char *hostname, int port, const char *path) {
    SSL_CTX *ctx;
    SSL *ssl;
    int sock;
    struct sockaddr_in server_addr;
    struct hostent *he;
    char buffer[BUFFER_SIZE];

    printf("Using HTTPS (encrypted with SSL/TLS)\n\n");

    // Initialize OpenSSL
    SSL_library_init();
    SSL_load_error_strings();
    OpenSSL_add_all_algorithms();

    // Create SSL context
    ctx = SSL_CTX_new(TLS_client_method());
    if (!ctx) {
        perror("SSL_CTX_new failed");
        return 1;
    }

    // Resolve hostname
    he = gethostbyname(hostname);
    if (!he) {
        perror("gethostbyname failed");
        SSL_CTX_free(ctx);
        return 1;
    }

    printf("Resolved IP: %s\n\n", inet_ntoa(*(struct in_addr *)he->h_addr_list[0]));

    // Create TCP socket
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket creation failed");
        SSL_CTX_free(ctx);
        return 1;
    }

    // Set up server address
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    memcpy(&server_addr.sin_addr, he->h_addr_list[0], he->h_length);

    // Connect to server
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Connection failed");
        close(sock);
        SSL_CTX_free(ctx);
        return 1;
    }

    printf("TCP Connected, performing SSL/TLS handshake...\n\n");

    // Create SSL connection
    ssl = SSL_new(ctx);
    SSL_set_fd(ssl, sock);
    SSL_set_tlsext_host_name(ssl, hostname);

    if (SSL_connect(ssl) <= 0) {
        ERR_print_errors_fp(stderr);
        SSL_free(ssl);
        close(sock);
        SSL_CTX_free(ctx);
        return 1;
    }

    printf("SSL/TLS handshake successful!\n\n");

    // Build HTTPS request
    char http_request[BUFFER_SIZE];
    snprintf(http_request, sizeof(http_request),
             "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n\r\n",
             path, hostname);

    printf("Sending HTTPS request...\n");
    printf("------- REQUEST -------\n%s", http_request);
    printf("------------------------\n\n");

    // Send HTTPS request
    if (SSL_write(ssl, http_request, strlen(http_request)) <= 0) {
        ERR_print_errors_fp(stderr);
        SSL_free(ssl);
        close(sock);
        SSL_CTX_free(ctx);
        return 1;
    }

    printf("Receiving response...\n");
    printf("------- RESPONSE -------\n");

    // Receive response
    int bytes_received;
    while ((bytes_received = SSL_read(ssl, buffer, sizeof(buffer) - 1)) > 0) {
        buffer[bytes_received] = '\0';
        printf("%s", buffer);
    }

    printf("\n------------------------\n");

    // Cleanup
    SSL_free(ssl);
    close(sock);
    SSL_CTX_free(ctx);
    EVP_cleanup();

    return 0;
}

int main() {
    char hostname[256];
    char path[256];
    char protocol[10];
    int port;

    printf("=== TCP Client - HTTP/HTTPS Web Server ===\n\n");

    // Get protocol from user
    printf("Enter protocol (http or https): ");
    fgets(protocol, sizeof(protocol), stdin);
    protocol[strcspn(protocol, "\n")] = '\0';

    // Get hostname from user
    printf("Enter hostname (e.g., google.com, example.com): ");
    fgets(hostname, sizeof(hostname), stdin);
    hostname[strcspn(hostname, "\n")] = '\0';

    // Get path from user
    printf("Enter path (e.g., / or /index.html): ");
    fgets(path, sizeof(path), stdin);
    path[strcspn(path, "\n")] = '\0';

    // If path is empty, use default root
    if (strlen(path) == 0) {
        strcpy(path, "/");
    }

    // Determine port based on protocol
    if (strcmp(protocol, "https") == 0 || strcmp(protocol, "HTTPS") == 0) {
        port = 443;
        printf("\nConnecting to %s:%d (HTTPS)...\n\n", hostname, port);
        return connect_https(hostname, port, path);
    } else {
        port = 80;
        printf("\nConnecting to %s:%d (HTTP)...\n\n", hostname, port);
        return connect_http(hostname, port, path);
    }
}
~~~

~~~(空)
$ make web_client_tcp 
cc -o web_client_tcp web_client_tcp.c -lssl -lcrypto

$ ./web_client_tcp 
=== TCP Client - HTTP/HTTPS Web Server ===

Enter protocol (http or https): https
Enter hostname (e.g., google.com, example.com): bilibili.com
Enter path (e.g., / or /index.html): /

Connecting to bilibili.com:443 (HTTPS)...

Using HTTPS (encrypted with SSL/TLS)

Resolved IP: 119.3.70.188

TCP Connected, performing SSL/TLS handshake...

SSL/TLS handshake successful!

Sending  HTTPS request...
------- REQUEST -------
GET / HTTP/1.1
Host: bilibili.com
Connection: close

------------------------

Receiving response...
------- RESPONSE -------
HTTP/1.1 301 Moved Permanently
Server: Tengine
Date: Fri, 21 Nov 2025 08:34:49 GMT
Content-Type: text/html
Content-Length: 239
Connection: close
Location: https://www.bilibili.com/

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr/>Powered by Tengine<hr><center>tengine</center>
</body>
</html>

------------------------
~~~

以上两个案例都需要保证主机网络环境正常。



### OSI 模型

~~~swift
┌─────────────────────────────┐  Layer 7 - 应用层
│ 应用层 (Application)        │  - 用户进程、应用协议（HTTP/SMTP/FTP/DNS）
└─────────────────────────────┘
┌─────────────────────────────┐  Layer 6 - 表示层
│ 表示层 (Presentation)       │  - 数据编码/解码、加密（SSL/TLS、MIME）
└─────────────────────────────┘
┌─────────────────────────────┐  Layer 5 - 会话层
│ 会话层 (Session)            │  - 会话/连接管理（RPC, NetBIOS 会话）
└─────────────────────────────┘											应用层
---------------------------------------------------------------------------------
┌─────────────────────────────┐  Layer 4 - 传输层
│ 传输层 (Transport)          │  - 端到端、可靠性（TCP/UDP）
└─────────────────────────────┘								 TCP/UDP/IPv4/IPv6/...
┌─────────────────────────────┐  Layer 3 - 网络层
│ 网络层 (Network)            │  - 路由、逻辑寻址（IP, ICMP, 路由协议）
└─────────────────────────────┘
---------------------------------------------------------------------------------
┌─────────────────────────────┐  Layer 2 - 数据链路层			   设备驱动程序和硬件
│ 数据链路层 (Data Link)      │  - 帧、MAC、错误检测（Ethernet, ARP）
└─────────────────────────────┘
┌─────────────────────────────┐  Layer 1 - 物理层
│ 物理层 (Physical)           │  - 物理媒介与信号（电缆、光纤、接口标准）
└─────────────────────────────┘

~~~

我们主要研究的是网络层和传输层，套接字编程的接口就是从顶上三层（网际协议应用层）进入传输层的接口。本书的焦点是：如何使用套接字编写使用TCP或UDP的网络应用程序。

Q: 为什么套接字提供的是从OSI模型的顶上三层进入传输层的接口？

A: 1.顶上三层处理具体的网络应用（如FTP、Telnet或HTTP）的所有细节，却对通信细节了解很少；底下四层对具体的网络应用了解很少，却处理所有的通信细节，发送数据，等待确认，给无序到达的数据排序，计算验证和，等等。 2.顶上三层通常构成所谓的用户进程user process，底下四层通常作为操作系统内核的一部分提供。

Unix与其他现代操作系统都提供分隔用户进程和内核的机制。由此可见第四层和第五层之间自然是构建API接口的位置。









