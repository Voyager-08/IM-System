# IM即时通讯系统

meChat 是一套基于 Qt 开发的完整即时通讯解决方案，包含 **桌面客户端 (meChatClient)** 与 **TCP 聊天服务器 (meChatServer)**。项目支持实时消息收发、好友管理、历史记录持久化及离线消息推送，采用 C++17 编写，实现了 UI、网络 I/O 与数据库操作的三线程异步分离架构。

## 📋 目录
- [功能特性](#-功能特性)
- [技术栈](#️-技术栈)
- [系统架构](#️-系统架构)
- [通信协议](#-通信协议)
- [数据库设计](#-数据库设计)
- [项目结构](#-项目结构)
- [构建与运行](#️-构建与运行)
- [核心亮点](#-核心亮点)

---

## ✨ 功能特性

### 客户端 (meChatClient)
- **用户系统**：注册、登录、历史登录记忆
- **实时通讯**：TCP 长连接 + 心跳保活，消息实时收发
- **好友管理**：搜索添加好友、删除好友、好友列表展示
- **消息系统**：文本消息发送/接收、本地历史消息秒级加载
- **界面交互**：页面切换动画、自定义圆角消息气泡淡入动画、右键菜单

### 服务器端 (meChatServer)
- **用户管理**：注册/登录验证、信息维护（昵称/邮箱/签名/性别）、在线状态管理、头像上传下载
- **好友系统**：添加/删除好友、模糊搜索（ID/昵称）、备注功能
- **消息系统**：实时私聊转发、离线消息存储与上线推送、消息持久化
- **连接管理**：心跳检测、断线自动清理、多客户端并发支持

---

## 🛠️ 技术栈

| 类别 | 客户端 (Client) | 服务器端 (Server) |
| :--- | :--- | :--- |
| **语言** | C++17 | C++17 |
| **框架** | Qt6 (Widgets) | Qt5 (Core, Network, Sql) |
| **网络** | QTcpSocket | QTcpServer / QTcpSocket |
| **数据库** | SQLite (本地缓存) | SQLite (持久化存储) |
| **构建** | CMake ≥ 3.16 | CMake ≥ 3.16 |
| **编译器** | MinGW | GCC / MSVC / MinGW |

---

## 🏗️ 系统架构

### 多线程模型
采用 **UI / 网络 / 数据库** 三线程分离设计，彻底避免界面卡顿与竞态问题：

```text
┌─────────────────────────────────────────────────┐
│                  UI 主线程                       │
│  MainWindow → LoginWindow → ChatWindow          │
│  (页面渲染 / 动画 / 用户交互)                     │
└───────────────┬────────────────┬────────────────┘
                │  信号槽         │  信号槽
                ▼                ▼
┌───────────────────────┐ ┌───────────────────────┐
│    网络线程            │ │    数据加载线程        │
│  NetworkManager       │ │  DataLoader           │
│  (TCP Socket / 心跳)   │ │  (SQLite 读写)        │
└───────────────────────┘ └───────────────────────┘
```

- 通过 `QObject::moveToThread` 实现线程迁移
- 线程间完全通过 Qt 信号槽通信，使用 `Qt::QueuedConnection` 保证安全
- 使用 `QMetaObject::invokeMethod` 进行跨线程方法调度

### Model/View/Delegate 分层渲染
- **Model**：继承 `QAbstractListModel`，自定义 Role 传递多字段数据
- **Delegate**：继承 `QStyledItemDelegate`，重写 `paint` 实现头像圆角裁剪与多行文本渲染
- **缓存优化**：使用 `QCache` / `QHash` 缓存头像，避免重复磁盘 I/O

---

## 📡 通信协议

基于 **TCP + JSON** 的自定义应用层协议，每条消息以换行符 `\n` 结尾。

### 消息类型总览

| 消息类型 | 方向 | 说明 |
| :--- | :--- | :--- |
| `register` / `register_result` | C↔S | 用户注册请求与响应 |
| `login` / `login_result` | C↔S | 登录验证请求与响应 |
| `logout` | C→S | 退出登录通知 |
| `user_info` | C↔S | 获取/返回用户详细信息 |
| `message` / `message_result` | C↔S | 实时消息发送与确认 |
| `heartbeat` / `heartbeat_response` | C↔S | 心跳保活机制 |
| `user_friends` / `friend_list` | C↔S | 加载好友列表 |
| `add_friend` / `add_friend_result` | C↔S | 添加好友请求与结果 |
| `delete_friend` / `delete_friend_result` | C↔S | 删除好友请求与结果 |
| `search_friend_list` | C↔S | 搜索好友（支持模糊匹配） |
| `avatar_file` / `avatar_data` | C↔S | 头像上传(Base64)与下发 |
| `离线消息` | S→C | 上线时推送历史未读消息 |

### 关键消息示例

**登录请求：**
```json
{"type": "login", "userId": "user001", "password": "hashed_pwd"}
```

**发送消息：**
```json
{
  "type": "message",
  "sender": "user001",
  "receiver": "user002",
  "content": "Hello!",
  "messageType": "text",
  "timestamp": "2026-06-30 12:00:00"
}
```

**离线消息推送：**
```json
{
  "type": "离线消息",
  "sender": "user003",
  "receiver": "user001",
  "content": "你有一条新消息",
  "messageType": "text",
  "timestamp": "2026-06-30 12:00:00"
}
```

---

## 💾 数据库设计 (Server)

服务器端使用 SQLite 持久化数据，包含以下四张核心表：

```sql
-- 用户表
CREATE TABLE users (
    user_Id TEXT PRIMARY KEY,
    user_Nick TEXT, password TEXT, email TEXT,
    created_at TEXT, motto TEXT, avatar_path TEXT, sex TEXT
);

-- 好友关系表
CREATE TABLE friendships (
    user_Id TEXT, friend_Id TEXT, friend_note TEXT,
    PRIMARY KEY (user_Id, friend_Id)
);

-- 聊天记录表
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sender_id TEXT, receiver_id TEXT, comment TEXT,
    message_type TEXT, datetime TEXT
);

-- 离线消息表
CREATE TABLE offline_messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_uuid TEXT, sender_id TEXT, receiver_id TEXT,
    content TEXT, message_type TEXT, datetime TEXT
);
```

> ⚠️ **头像存储路径**：默认接收路径为 `./images/avatar/{userId}.png`，可在 `chatserver.cpp` 中修改。

---

## 📂 项目结构

### 客户端 (src/)
```text
src/
├── main.cpp                    # 程序入口
├── core/                       # 核心界面模块
│   ├── main/mainwindow.*       #   主窗口（登录/聊天切换）
│   ├── login/                  #   登录注册模块
│   ├── chat/chatwindow.*       #   聊天主窗口
│   └── extra/messagebubble.*   #   自绘消息气泡控件
├── thread/                     # 多线程模块
│   ├── network/networkmanager.*#   网络通信线程
│   └── dataLoad/dataloader.*   #   数据加载线程
├── modelView/                  # Model/View/Delegate 层
├── sql/database.*              # SQLite 本地封装
└── custom/                     # 自定义组件 (struct.h, 输入框, 可点击标签)
```

### 服务器端 (meChatServer/)
```text
meChatServer/
├── src/
│   ├── main.cpp           # 入口与端口配置(默认6452)
│   ├── chatserver.*       # 服务器核心逻辑
│   └── database.*         # 数据库操作封装
├── make.sh                # Linux 一键编译脚本
└── CMakeLists.txt
```

---

## ⚙️ 构建与运行

### 环境要求
- CMake ≥ 3.16
- Qt ≥ 6.5 (客户端) / Qt 5.x (服务器)
- C++17 兼容编译器

### 客户端编译
```bash
mkdir build && cd build
cmake .. -G "MinGW Makefiles"
cmake --build .
./bin/meChatClient.exe
```

### 服务器编译

**Linux (推荐):**
```bash
chmod +x make.sh && ./make.sh
```

**手动构建:**
```bash
mkdir build && cd build
cmake .. && make -j$(nproc)
./meChatServer
```

**Windows:**
```powershell
mkdir build; cd build
cmake ..
cmake --build . --config Release
.\Release\meChatServer.exe
```

### 服务器配置
- **端口**：默认 `6452`，在 `src/main.cpp` 中修改
- **数据库**：确保 `../sql/sqlite/mechat.sqlite` 存在且表结构正确

---

## 🌟 核心亮点

1.  **自研 TCP 应用层协议**：JSON 消息体 + `msgType` 路由分发，支持 10+ 种消息类型，扩展性强
2.  **三线程异步架构**：UI / 网络 / 数据库完全物理隔离，信号槽解耦，零竞态、零卡顿
3.  **自定义消息气泡**：QWidget 自绘圆角 + 三角箭头，配合 `QPropertyAnimation` 实现丝滑淡入
4.  **高性能列表渲染**：Model/View/Delegate 分层 + 头像内存缓存，万条消息流畅滚动
5.  **完整的离线消息机制**：服务端暂存 + 上线批量推送，消息不丢失
6.  **本地持久化**：客户端三表设计 + 参数化查询，历史消息秒级检索

---
