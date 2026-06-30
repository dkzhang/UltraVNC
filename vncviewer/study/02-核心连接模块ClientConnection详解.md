# ClientConnection 核心连接模块详解

## 1. 模块概述

ClientConnection 是 UltraVNC Viewer 的核心类，负责管理与 VNC 服务器的完整连接生命周期。该类继承自 `omni_thread`，在独立线程中运行网络通信和协议处理。

### 1.1 类定义位置
- **头文件**: `ClientConnection.h`
- **实现文件**: `ClientConnection.cpp` 及其分文件

### 1.2 类继承关系
```
omni_thread (基类 - 跨平台线程类)
    │
    └──▶ ClientConnection
            │
            ├── 友元类: FileTransfer
            ├── 友元类: TextChat
            └── 组合: VNCOptions, KeyMap, KeyMapJap
```

---

## 2. 类结构详解

### 2.1 核心成员变量

```cpp
class ClientConnection : public omni_thread
{
public:
    // ========== 连接状态变量 ==========
    HWND m_hSessionDialog;          // 会话对话框句柄
    int m_port;                     // 目标端口
    int m_proxyport;                // 代理端口
    TCHAR m_host[MAX_HOST_NAME_LEN]; // 主机名
    TCHAR m_proxyhost[MAX_HOST_NAME_LEN]; // 代理主机
    ConnectionType m_connectionType; // 连接类型
    
    // ========== 服务器信息 ==========
    rfbServerInitMsg m_si;          // 服务器初始化消息
    TCHAR *m_desktopName;           // 桌面名称
    int m_cliwidth, m_cliheight;    // 客户端区域大小
    
    // ========== 套接字和网络 ==========
    SOCKET m_sock;                  // TCP 套接字
    char m_QueueBuffer[G_SENDBUFFER+1]; // 发送队列缓冲区
    DWORD m_nQueueBufferLength;     // 队列缓冲区长度
    
    // ========== 窗口句柄 ==========
    HWND m_hwndMain;                // 主窗口句柄
    HWND m_hwndcn;                  // 连接窗口
    HWND m_hwndTB;                  // 工具栏窗口
    HWND m_hwndStatus;              // 状态栏窗口
    HWND m_TrafficMonitor;          // 流量监控窗口
    
    // ========== 选项和映射 ==========
    VNCOptions *m_opts;             // 连接选项
    VNCOptions m_optsCopy;          // 选项副本
    KeyMap *m_keymap;               // 键盘映射
    KeyMapJap *m_keymapJap;        // 日语键盘映射
    
    // ========== 扩展功能 ==========
    FileTransfer *m_pFileTransfer;  // 文件传输对象
    TextChat *m_pTextChat;          // 文本聊天对象
    CDSMPlugin *m_pDSMPlugin;       // DSM 插件对象
    
    // ========== 缓冲区管理 ==========
    char *m_netbuf;                 // 网络缓冲区
    UINT m_netbufsize;               // 缓冲区大小
    unsigned char *m_zlibbuf;        // Zlib 解压缓冲区
    int m_zlibbufsize;              // Zlib 缓冲区大小
    
    // ========== 位图和渲染 ==========
    HDC m_hBitmapDC;                // 位图设备上下文
    VOID *m_DIBbits;                // DIB 位数据指针
    BYTE *m_DIBbitsCache;           // 缓存位数据
    
    // ========== 帧缓冲 ==========
    int m_hScrollPos, m_vScrollPos; // 滚动位置
    int m_hScrollMax, m_vScrollMax; // 滚动范围
    
    // ========== 像素格式 ==========
    rfbPixelFormat m_myFormat;      // 本地像素格式
    rfbPixelFormat m_pendingFormat;  // 待切换格式
    int m_majorVersion, m_minorVersion; // 协议版本
    
    // ========== 状态标志 ==========
    bool m_running;                 // 运行状态
    bool m_bKillThread;             // 线程终止标志
    bool m_serverInitiated;         // 服务器发起连接
    bool m_dormant;                 // 休眠状态
    
    // ========== 统计信息 ==========
    __int64 m_BytesSend;           // 已发送字节数
    __int64 m_BytesRead;           // 已接收字节数
    unsigned int kbitsPerSecond;   // 当前带宽
    unsigned int avg_kbitsPerSecond; // 平均带宽
    
    // ========== 互斥锁 ==========
    omni_mutex m_bufferMutex;       // 缓冲区锁
    omni_mutex m_zlibBufferMutex;   // Zlib 缓冲区锁
    omni_mutex m_bitmapdcMutex;     // 位图锁
    omni_mutex m_clipMutex;         // 剪贴板锁
    omni_mutex m_writeMutex;        // 写操作锁
    omni_mutex m_sockMutex;         // 套接字锁
    omni_mutex m_cursorMutex;       // 光标锁
    omni_mutex m_readMutex;         // 读操作锁
};
```

### 2.2 构造函数变体

```cpp
// 1. 标准构造函数 - 创建新连接
ClientConnection(VNCviewerApp *pApp);

// 2. 从套接字创建 - 监听模式接受连接
ClientConnection(VNCviewerApp *pApp, SOCKET sock);

// 3. 从主机端口创建 - 命令行连接
ClientConnection(VNCviewerApp *pApp, LPTSTR host, int port);

// 4. 从配置文件创建 - 加载保存的连接
ClientConnection(VNCviewerApp *pApp, LPTSTR configFile);
```

---

## 3. 连接生命周期

### 3.1 连接建立流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          连接建立完整流程                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Init() - 初始化                                                      │
│     - 设置默认参数                                                      │
│     - 初始化互斥锁                                                      │
│     - 创建 GDI 对象                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. GetConnectDetails() - 获取连接详情                                   │
│     - 显示会话对话框                                                    │
│     - 解析主机地址                                                      │
│     - 加载连接配置                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        ┌─────────────────────┐         ┌─────────────────────┐
        │  3a. Connect()      │         │  3b. ConnectProxy() │
        │     直接 TCP 连接   │         │     通过代理连接     │
        └──────────┬──────────┘         └──────────┬──────────┘
                   │                               │
                   └───────────────┬───────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. SetSocketOptions() - 设置套接字选项                                   │
│     - TCP_NODELAY (禁用 Nagle 算法)                                      │
│     - SO_KEEPALIVE (保持连接)                                            │
│     - 发送/接收超时                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  5. LoadDSMPlugin() - 加载 DSM 插件 (如果启用)                            │
│     - 加载插件 DLL                                                      │
│     - 初始化插件接口                                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  6. NegotiateProtocolVersion() - 协商协议版本                             │
│     - 接收服务器版本                                                    │
│     - 选择共同支持的最高版本                                             │
│     - 发送客户端版本                                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  7. Authenticate() - 身份认证                                            │
│     - 接收服务器支持的安全类型                                           │
│     - 根据类型执行相应认证流程                                           │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  支持的认证类型:                                              │    │
│     │  - rfbSecTypeNone          (无认证)                          │    │
│     │  - rfbSecTypeVncAuth       (VNC 密码认证)                    │    │
│     │  - rfbSecTypeTight         (TightVNC 安全类型)               │    │
│     │  - rfbSecTypeVeNCrypt      (VeNCrypt 加密)                   │    │
│     │  - rfbSecTypeMsLogonI/II/III (MS-Logon)                     │    │
│     │  - rfbSecTypeRSAAES        (RSA-AES 加密)                   │    │
│     └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  8. ReadServerInit() - 读取服务器初始化信息                               │
│     - 帧缓冲尺寸                                                        │
│     - 像素格式                                                          │
│     - 桌面名称                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  9. CreateLocalFramebuffer() - 创建本地帧缓冲                             │
│     - 分配 DIB 位图内存                                                 │
│     - 创建兼容的 DC                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  10. SetupPixelFormat() - 设置像素格式                                    │
│      - 确定颜色深度                                                      │
│      - 配置 RGB 掩码                                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  11. SetFormatAndEncodings() - 发送格式和编码设置                          │
│      - 发送像素格式                                                     │
│      - 发送支持的编码列表                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  12. SendClientInit() - 发送客户端初始化消息                              │
│      - 共享标志                                                         │
│      - 请求初始帧缓冲                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  13. CreateDisplay() - 创建显示窗口                                       │
│      - 创建主窗口                                                       │
│      - 创建工具栏                                                       │
│      - 创建状态栏                                                       │
│      - 设置窗口位置和大小                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  14. Run() - 进入主循环                                                   │
│      - 启动网络线程                                                     │
│      - 处理窗口消息                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 主循环处理 (run_undetached)

```cpp
void* ClientConnection::run_undetached(void* arg)
{
    // 主网络处理循环
    while (m_running && !m_bKillThread) {
        // 1. 检查是否需要请求帧缓冲更新
        if (NeedFramebufferUpdate()) {
            SendAppropriateFramebufferUpdateRequest(false);
        }
        
        // 2. 读取服务器消息
        ReadScreenUpdate();
        
        // 3. 处理 Keep-Alive
        if (m_server_wants_keepalives) {
            SendKeepAlive(false, false);
        }
    }
    return NULL;
}
```

---

## 4. 网络通信机制

### 4.1 数据发送

```cpp
// 写入数据到网络 - 核心函数
void ClientConnection::WriteExact(char *buf, int bytes)
{
    // 1. 检查连接状态
    if (m_sock == INVALID_SOCKET) {
        throw SocketExc("Socket invalid");
    }
    
    // 2. 加锁保护写操作
    omni_mutex_lock ml(m_writeMutex);
    
    // 3. 可选: 通过 DSM 插件转换数据
    if (m_fUsePlugin && m_pPluginInterface) {
        // 数据加密/转换
        BYTE* transformed = TransformBuffer((BYTE*)buf, bytes, &newLen);
        buf = (char*)transformed;
        bytes = newLen;
    }
    
    // 4. 循环发送直到完成
    int pos = 0;
    while (pos < bytes) {
        int sent = send(m_sock, buf + pos, bytes - pos, 0);
        if (sent == SOCKET_ERROR) {
            int err = WSAGetLastError();
            if (err != WSAEWOULDBLOCK) {
                throw SocketExc("Send failed");
            }
            // 等待可写
            WaitForWritable();
        } else {
            pos += sent;
            m_BytesSend += sent;
        }
    }
}

// 队列发送 - 合并小数据包
void ClientConnection::WriteExactQueue(char *buf, int bytes)
{
    omni_mutex_lock ml(m_writeMutex);
    
    // 如果数据量小且队列未满，加入队列
    if (bytes <= G_SENDBUFFER - m_nQueueBufferLength) {
        memcpy(m_QueueBuffer + m_nQueueBufferLength, buf, bytes);
        m_nQueueBufferLength += bytes;
        return;
    }
    
    // 队列已满，先刷新
    FlushWriteQueue();
    
    // 新数据加入队列
    memcpy(m_QueueBuffer, buf, bytes);
    m_nQueueBufferLength = bytes;
}
```

### 4.2 数据接收

```cpp
// 精确读取指定字节数
void ClientConnection::ReadExact(char *buf, int bytes)
{
    omni_mutex_lock ml(m_readMutex);
    
    int pos = 0;
    while (pos < bytes) {
        int read = recv(m_sock, buf + pos, bytes - pos, 0);
        
        if (read == SOCKET_ERROR) {
            int err = WSAGetLastError();
            if (err != WSAEWOULDBLOCK) {
                throw SocketExc("Recv failed");
            }
            // 等待可读
            WaitForReadable();
        } else if (read == 0) {
            // 连接关闭
            throw SocketExc("Connection closed");
        } else {
            pos += read;
            m_BytesRead += read;
        }
    }
    
    // DSM 插件解密
    if (m_fUsePlugin && m_pPluginInterface) {
        BYTE* restored = RestoreBufferStep1((BYTE*)buf, bytes, &newLen);
        restored = RestoreBufferStep2(restored, newLen, &finalLen);
        // ... 处理解密后的数据
    }
}
```

---

## 5. 帧缓冲更新机制

### 5.1 请求策略

```cpp
// 发送帧缓冲更新请求
void ClientConnection::SendFramebufferUpdateRequest(
    int x, int y, int w, int h, bool incremental)
{
    rfbFramebufferUpdateRequestMsg fur;
    
    fur.type = rfbFramebufferUpdateRequest;
    fur.incremental = incremental ? 1 : 0;
    fur.x = Swap16IfLE(x);
    fur.y = Swap16IfLE(y);
    fur.w = Swap16IfLE(w);
    fur.h = Swap16IfLE(h);
    
    WriteExact((char*)&fur, sz_rfbFramebufferUpdateRequestMsg);
}

// 智能请求策略
void ClientConnection::SendAppropriateFramebufferUpdateRequest(bool bAsync)
{
    if (m_pendingFormatChange) {
        // 像素格式改变时请求完整更新
        SendFullFramebufferUpdateRequest(bAsync);
    } else {
        // 增量更新
        SendIncrementalFramebufferUpdateRequest(bAsync);
    }
}
```

### 5.2 更新消息处理

```cpp
void ClientConnection::ReadScreenUpdate()
{
    rfbServerToClientMsg msg;
    ReadExact((char*)&msg, 1);
    
    switch (msg.type) {
        case rfbFramebufferUpdate:
            HandleFramebufferUpdate();
            break;
            
        case rfbSetColourMapEntries:
            // 调色板更新 (罕见)
            break;
            
        case rfbBell:
            ReadBell();
            break;
            
        case rfbServerCutText:
            ReadServerCutText();
            break;
            
        case rfbResizeFrameBuffer:
            // 帧缓冲大小改变
            ReadNewFBSize(&msg);
            break;
            
        case rfbFileTransfer:
            // 文件传输消息
            m_pFileTransfer->ProcessFileTransferMsg();
            break;
            
        case rfbTextChat:
            // 文本聊天消息
            m_pTextChat->ProcessTextChatMsg();
            break;
            
        case rfbKeepAlive:
            // Keep-Alive 响应
            break;
            
        default:
            throw ProtocolExc("Unknown message type");
    }
}
```

### 5.3 矩形编码分发

```cpp
void ClientConnection::HandleFramebufferUpdate()
{
    rfbFramebufferUpdateMsg msg;
    ReadExact((char*)&msg, sz_rfbFramebufferUpdateMsg - 1);
    
    msg.nRects = Swap16IfLE(msg.nRects);
    
    // 处理每个矩形
    for (int i = 0; i < msg.nRects; i++) {
        rfbFramebufferUpdateRectHeader rect;
        ReadExact((char*)&rect, sz_rfbFramebufferUpdateRectHeader);
        
        rect.encoding = Swap32IfLE(rect.encoding);
        rect.r.x = Swap16IfLE(rect.r.x);
        rect.r.y = Swap16IfLE(rect.r.y);
        rect.r.w = Swap16IfLE(rect.r.w);
        rect.r.h = Swap16IfLE(rect.r.h);
        
        // 根据编码类型分发处理
        switch (rect.encoding) {
            case rfbEncodingRaw:
                ReadRawRect(&rect);
                break;
                
            case rfbEncodingCopyRect:
                ReadCopyRect(&rect);
                break;
                
            case rfbEncodingRRE:
                ReadRRERect(&rect);
                break;
                
            case rfbEncodingCoRRE:
                ReadCoRRERect(&rect);
                break;
                
            case rfbEncodingHextile:
                ReadHextileRect(&rect);
                break;
                
            case rfbEncodingZlib:
                ReadZlibRect(&rect, false);
                break;
                
            case rfbEncodingTight:
                ReadTightRect(&rect, false);
                break;
                
            case rfbEncodingZstd:
                ReadZlibRect(&rect, true);
                break;
                
            case rfbEncodingUltra:
                ReadUltraRect(&rect);
                break;
                
            case rfbEncodingUltra2:
                ReadUltra2Rect(&rect);
                break;
                
            case rfbEncodingZlibHex:
                ReadZlibHexRect(&rect, false);
                break;
                
            // ... 其他编码类型
        }
    }
}
```

---

## 6. 输入事件处理

### 6.1 鼠标事件

```cpp
void ClientConnection::SendPointerEvent(int x, int y, int buttonMask)
{
    rfbPointerEventMsg pe;
    
    pe.type = rfbPointerEvent;
    pe.buttonMask = buttonMask;
    pe.x = Swap16IfLE(x);
    pe.y = Swap16IfLE(y);
    
    WriteExact((char*)&pe, sz_rfbPointerEventMsg);
    
    // 记录最后位置用于增量更新
    oldPointerX = x;
    oldPointerY = y;
    oldButtonMask = buttonMask;
}

// 鼠标轮事件处理
void ClientConnection::ProcessMouseWheel(int delta)
{
    int buttonMask = oldButtonMask;
    
    if (delta < 0) {
        buttonMask |= 8;  // 向下滚动
    } else {
        buttonMask |= 16; // 向上滚动
    }
    
    SendPointerEvent(oldPointerX, oldPointerY, buttonMask);
    SendPointerEvent(oldPointerX, oldPointerY, oldButtonMask);
}
```

### 6.2 键盘事件

```cpp
void ClientConnection::SendKeyEvent(CARD32 key, bool down)
{
    rfbKeyEventMsg ke;
    
    ke.type = rfbKeyEvent;
    ke.down = down ? 1 : 0;
    ke.key = Swap32IfLE(key);
    
    WriteExact((char*)&ke, sz_rfbKeyEventMsg);
}

// 窗口消息处理
void ClientConnection::ProcessKeyEvent(int virtkey, DWORD keyData)
{
    // 通过 KeyMap 转换为 X11 键码
    m_keymap->PCtoX(virtkey, keyData, this);
}
```

---

## 7. 分文件功能说明

### 7.1 ClientConnectionFile.cpp
处理文件相关的连接操作：
- `LoadConnection()` - 加载连接配置文件
- `SaveConnection()` - 保存连接配置
- 配置文件解析和序列化

### 7.2 ClientConnectionFullScreen.cpp
全屏模式处理：
- `SetFullScreenMode()` - 切换全屏模式
- `RealiseFullScreenMode()` - 实现全屏
- `BorderlessMode()` - 无边框模式
- `BumpScroll()` - 边缘滚动

### 7.3 ClientConnectionClipboard.cpp
剪贴板同步：
- `ProcessLocalClipboardChange()` - 处理本地剪贴板变化
- `UpdateRemoteClipboard()` - 更新远程剪贴板
- `ReadServerCutText()` - 读取服务器剪贴板文本
- 扩展剪贴板格式支持

### 7.4 ClientConnectionCursor.cpp
光标处理：
- `ReadCursorShape()` - 读取光标形状
- `ReadCursorPos()` - 读取光标位置 (CursorPos 扩展)
- `SoftCursorLockArea()` - 软光标锁定
- `SoftCursorDraw()` - 绘制软光标

### 7.5 ClientConnectionCacheRect.cpp
缓存矩形处理：
- `ReadCacheRect()` - 读取缓存矩形
- `SaveArea()` - 保存区域到缓存
- `RestoreArea()` - 从缓存恢复区域
- `ClearCache()` - 清除缓存

---

## 8. 线程安全设计

### 8.1 互斥锁使用

```cpp
// 读操作加锁
void ClientConnection::ReadExact(char *buf, int bytes)
{
    omni_mutex_lock ml(m_readMutex);
    // ... 实际读操作
}

// 写操作加锁
void ClientConnection::WriteExact(char *buf, int bytes)
{
    omni_mutex_lock ml(m_writeMutex);
    // ... 实际写操作
}

// 位图操作加锁
void ClientConnection::DoBlit()
{
    omni_mutex_lock ml(m_bitmapdcMutex);
    // ... 位图渲染
}
```

### 8.2 线程同步事件

```cpp
HANDLE KillEvent;           // 线程终止事件
HANDLE KillUpdateThreadEvent; // 更新线程终止事件

// 等待线程终止
void ClientConnection::KillThread()
{
    m_bKillThread = true;
    SetEvent(KillEvent);
    WaitForSingleObject(handle(), INFINITE);
}
```

---

## 9. 错误处理与异常

### 9.1 异常类定义

```cpp
class ClientConnection {
public:
    // 内部异常类
    class UserCancelExc {};     // 用户取消
    class AuthenticationExc {}; // 认证失败
    class SocketExc {};         // 套接字错误
    class ProtocolExc {};        // 协议错误
    class Fatal {};              // 致命错误
};
```

### 9.2 异常处理流程

```cpp
void ClientConnection::Run()
{
    try {
        DoConnection();
    }
    catch (UserCancelExc&) {
        // 用户取消，正常退出
        vnclog.Print(0, "User cancelled connection\n");
    }
    catch (AuthenticationExc&) {
        // 认证失败
        MessageBox(NULL, "Authentication failed", "Error", MB_ICONERROR);
    }
    catch (SocketExc& e) {
        // 网络错误
        vnclog.Print(0, "Socket error: %s\n", e.what());
    }
    catch (ProtocolExc& e) {
        // 协议错误
        vnclog.Print(0, "Protocol error: %s\n", e.what());
    }
    catch (...) {
        // 未知错误
        vnclog.Print(0, "Unknown error\n");
    }
    
    // 清理资源
    CloseWindows();
    DeregisterConnection();
}
```

---

## 10. 性能优化要点

### 10.1 发送缓冲区合并
```cpp
#define G_SENDBUFFER 2904  // 优化为 MTU 大小

// 小数据包合并发送
WriteExactQueue(buf, len);  // 入队
FlushWriteQueue();          // 批量发送
```

### 10.2 鼠标事件节流
```cpp
struct PendingMouseMove {
    DWORD dwMinimumMouseMoveInterval; // 最小间隔 (150ms)
    bool ShouldThrottle(bool bMouseKeyDown) {
        return (GetTickCount() - dwLastSentMouseMove) 
            < (bMouseKeyDown ? (dwMinimumMouseMoveInterval / 2) 
                            : dwMinimumMouseMoveInterval);
    }
};
```

### 10.3 帧率控制
```cpp
// FPS 计数器控制更新频率
Fps fps;
if (fps.ShouldRequestUpdate()) {
    SendAppropriateFramebufferUpdateRequest(false);
}
```

---

## 11. 总结

ClientConnection 是一个复杂但设计清晰的类，主要特点：

1. **模块化设计**: 通过分文件组织不同功能
2. **线程安全**: 使用互斥锁保护共享资源
3. **可扩展性**: 支持多种编码和安全认证
4. **性能优化**: 缓冲区合并、事件节流、帧率控制
5. **错误处理**: 完善的异常处理机制

理解 ClientConnection 是掌握 UltraVNC Viewer 架构的关键。
