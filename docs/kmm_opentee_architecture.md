# KMM + Open-TEE 模拟器架构详解

## 一、总体架构概览

Open-TEE 是一个符合 Global Platform TEE 规范的虚拟 TEE 实现。当用作 KMM（密钥管理模块）模拟器时，整体架构如下：

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              系统层面 (3+ 进程)                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌──────────────┐          ┌──────────────┐          ┌──────────────────────────────┐  │
│  │ tee_manager  │◄────────►│tee_launcher  │◄────────►│    kmmta.so 进程 (fork 出来)  │  │
│  │   (主进程)    │ socket   │ (fork 守护进程)│ socket   │                              │  │
│  └──────┬───────┘          └──────────────┘          └──────────────┬─────────────────┘  │
│         │                                                            │                   │
│         │                                                            │                   │
│         │  Unix Socket                                               │  双线程模型        │
│         │  (CA 连接)                                                 │                    │
│         ▼                                                            ▼                   │
│  ┌──────────────┐                                          ┌─────────────────────┐      │
│  │   kmmca.so   │                                          │  IO 线程 (主线程)   │◄─────┤
│  │  (CA 应用)   │                                          │  ta_process_loop()  │      │
│  │  链接 libtee │                                          │  - epoll 监听 socket│      │
│  └──────────────┘                                          │  - 接收消息入队列    │      │
│                                                            └──────────┬──────────┘      │
│                                                                       │ signal          │
│                                                                       ▼ (pthread_cond)  │
│                                                            ┌─────────────────────┐      │
│                                                            │ Internal 线程       │      │
│                                                            │ ta_internal_thread()│      │
│                                                            │ - 从队列取消息      │      │
│                                                            │ - 调用 TA_xxx 入口  │      │
│                                                            │ - 执行 kmmta 逻辑   │      │
│                                                            └─────────────────────┘      │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

## 二、核心组件说明

### 2.1 系统级进程（由 opentee-engine 启动）

| 进程 | 可执行文件 | 源码位置 | 作用 |
|------|-----------|---------|------|
| **tee_manager** | opentee-engine | `emulator/manager/` | TEE 主控进程，监听 Unix Socket，接受 CA 连接，管理会话生命周期 |
| **tee_launcher** | opentee-engine | `emulator/launcher/` | TA 进程工厂，专门负责 fork 新的 TA 进程 |
| **kmmta 进程** | kmmta.so | `emulator/launcher/ta_process.c` | 由 launcher fork 的独立进程，加载 kmmta.so |

### 2.2 CA 侧组件

| 组件 | 类型 | 说明 |
|------|------|------|
| **kmmca.so** | 用户实现 | KMM 的客户端应用 |
| **libtee.so** | Open-TEE 提供 | TEE Client API 实现 (`libtee/src/tee_client_api.c`) |

### 2.3 TA 侧组件（每个 TA 进程内部）

| 线程 | 函数入口 | 源码位置 | 职责 |
|------|---------|---------|------|
| **IO 线程** | `ta_process_loop()` | `emulator/launcher/ta_process.c` | 通过 epoll 监听 socket，接收来自 manager 的消息，放入任务队列 |
| **Internal 线程** | `ta_internal_thread()` | `emulator/launcher/ta_internal_thread.c` | 从队列取出任务，调用 TA 入口函数执行实际逻辑 |

## 三、关键代码文件索引

### 3.1 通信协议
- `emulator/include/com_protocol.h` - 消息格式定义
- `libtee/include/com_protocol.h` - CA 侧通信协议

### 3.2 CA 侧实现
- `libtee/src/tee_client_api.c` - TEEC_xxx API 实现 (TEEC_InitializeContext, TEEC_OpenSession, TEEC_InvokeCommand 等)

### 3.3 Manager 进程
- `emulator/manager/mainloop.c` - manager 主循环
- `emulator/manager/io_thread.c` - IO 处理
- `emulator/manager/logic_thread.c` - 逻辑处理

### 3.4 Launcher 进程
- `emulator/launcher/launcher_mainloop.c` - launcher 主循环，`lib_main_loop()` 函数

### 3.5 TA 进程
- `emulator/launcher/ta_process.c` - TA 进程入口，`ta_process_loop()`
- `emulator/launcher/ta_io_thread.c` - IO 线程实现
- `emulator/launcher/ta_internal_thread.c` - Internal 线程实现

### 3.6 TA 加载与接口
- `emulator/launcher/dynamic_loader.c` - TA 动态加载，使用 `dlopen()` 加载 .so
- `emulator/internal_api/tee_ta_interface.h` - TA 入口函数定义 (TA_CreateEntryPoint, TA_OpenSessionEntryPoint 等)

### 3.7 GP Internal API 实现
- `emulator/internal_api/crypto/` - 加密 API 实现
- `emulator/internal_api/storage/` - 存储 API 实现
- `emulator/internal_api/tee_*.c` - 其他 GP API 实现

## 四、消息流转详解

### 4.1 会话建立流程

```
kmmca.so                    libtee.so                  tee_manager              tee_launcher           kmmta 进程
    │                          │                           │                      │                    │
    │ 1. TEEC_InitializeContext│                           │                      │                    │
    │─────────────────────────►│                           │                      │                    │
    │                          │ 2. connect()              │                      │                    │
    │                          │──────────────────────────►│                      │                    │
    │                          │                           │                      │                    │
    │ 3. TEEC_OpenSession      │                           │                      │                    │
    │─────────────────────────►│                           │                      │                    │
    │                          │ 4. send OPEN_SESSION      │                      │                    │
    │                          │──────────────────────────►│                      │                    │
    │                          │                           │ 5. fork/exec         │                    │
    │                          │                           │─────────────────────►│                    │
    │                          │                           │                      │ 6. clone()         │
    │                          │                           │                      │───────────────────►│
    │                          │                           │                      │                    │
    │                          │                           │ 7. send socket fd    │                    │
    │                          │                           │◄─────────────────────│                    │
    │                          │                           │                      │                    │
    │                          │                           │ 8. 直接与 TA 通信    │                    │
    │                          │◄──────────────────────────│                      │                    │
    │◄─────────────────────────│                           │                      │                    │
```

### 4.2 命令调用流程

```
kmmca.so           libtee.so           tee_manager           kmmta 进程 (IO 线程)      kmmta 进程 (Internal 线程)
    │                  │                     │                          │                          │
    │  TEEC_InvokeCmd  │                     │                          │                          │
    │─────────────────►│                     │                          │                          │
    │                  │  send INVOKE_CMD    │                          │                          │
    │                  │────────────────────►│                          │                          │
    │                  │                     │  转发给 TA               │                          │
    │                  │                     │─────────────────────────►│                          │
    │                  │                     │                          │  epoll 收到消息          │
    │                  │                     │                          │  add_task_todo_queue()   │
    │                  │                     │                          │  pthread_cond_signal()   │
    │                  │                     │                          │─────────────────────────►│
    │                  │                     │                          │                          │  pthread_cond_wait()
    │                  │                     │                          │                          │  被唤醒
    │                  │                     │                          │                          │  调用 TA_InvokeCommandEntryPoint()
    │                  │                     │                          │                          │  (kmmta 逻辑执行)
    │                  │                     │                          │◄─────────────────────────│
    │                  │                     │  返回结果               │  reply_to_manager()      │
    │                  │                     │◄─────────────────────────│                          │
    │                  │◄────────────────────│                          │                          │
    │◄─────────────────│                     │                          │                          │
```

### 4.3 通信协议消息类型

```c
// emulator/include/com_protocol.h

#define COM_MSG_NAME_CA_INIT_CONTEXT    0x01
#define COM_MSG_NAME_OPEN_SESSION       0x02
#define COM_MSG_NAME_CREATED_TA         0x03
#define COM_MSG_NAME_INVOKE_CMD         0x04
#define COM_MSG_NAME_CLOSE_SESSION      0x05
#define COM_MSG_NAME_CA_FINALIZ_CONTEXT 0x06
#define COM_MSG_NAME_PROC_STATUS_CHANGE 0x07
#define COM_MSG_NAME_FD_ERR             0x08
#define COM_MSG_NAME_ERROR              0x09
#define COM_MSG_NAME_OPEN_SHM_REGION    0x0C
#define COM_MSG_NAME_UNLINK_SHM_REGION  0x0D
#define COM_MSG_NAME_INVOKE_MGR_CMD     0x0F
```

## 五、KMM 集成要点

### 5.1 kmmta.so 需要实现的接口

根据 `emulator/internal_api/tee_ta_interface.h`：

```c
// 必须实现的 5 个入口函数
TEE_Result TA_EXPORT TA_CreateEntryPoint(void);
void       TA_EXPORT TA_DestroyEntryPoint(void);
TEE_Result TA_EXPORT TA_OpenSessionEntryPoint(uint32_t paramTypes, TEE_Param params[4], void **sessionContext);
void       TA_EXPORT TA_CloseSessionEntryPoint(void *sessionContext);
TEE_Result TA_EXPORT TA_InvokeCommandEntryPoint(void *sessionContext, uint32_t commandID, uint32_t paramTypes, TEE_Param params[4]);
```

### 5.2 kmmta.so 可使用的 GP API

- **存储**: `TEE_CreatePersistentObject`, `TEE_ReadObjectData`, `TEE_WriteObjectData`, `TEE_CloseObject`
- **加密**: `TEE_AllocateOperation`, `TEE_CipherInit`, `TEE_CipherUpdate`, `TEE_DigestDoFinal`
- **内存**: `TEE_Malloc`, `TEE_Free`
- **随机数**: `TEE_GenerateRandom`
- **其他**: `TEE_GetSystemTime`, `TEE_InvokeMGRCommand`

### 5.3 目录结构

```
/opt/OpenTee/
├── bin/
│   └── opentee-engine        # TEE 引擎主程序
├── lib/
│   ├── libtee.so             # CA 侧 Client API
│   ├── libManagerApi.so      # Manager 组件
│   ├── libLauncherApi.so     # Launcher 组件
│   ├── libOpenTeeInternalApi.so  # TA 侧 Internal API
│   └── TAs/
│       └── kmmta.so          # KMM 的 TA 实现
└── include/
    ├── tee_client_api.h      # CA API 头文件
    └── tee_internal_api.h    # TA API 头文件
```

## 六、启动流程

```bash
# 1. 启动 TEE 引擎（后台运行 tee_manager 和 tee_launcher）
sudo /opt/OpenTee/bin/opentee-engine -f

# 2. 运行 KMM CA 应用
./kmmca_app
   │
   ├── 链接 libtee.so
   ├── 调用 TEEC_InitializeContext() → 连接 tee_manager
   ├── 调用 TEEC_OpenSession() → 触发 tee_launcher fork kmmta 进程
   └── 调用 TEEC_InvokeCommand() → 与 kmmta 进程通信
```

## 七、设计优势

1. **进程隔离**：每个 TA 是独立进程，一个 TA 崩溃不影响其他 TA 或 Manager
2. **线程分工明确**：
   - IO 线程专职网络通信，保证响应及时
   - Internal 线程专职业务逻辑，避免阻塞
3. **符合 GP 规范**：CA 和 TA 接口均符合 Global Platform TEE 规范
4. **调试友好**：TA 作为独立进程，可用 gdb 直接 attach 调试

## 八、注意事项

1. **socket 路径**：默认使用 `/tmp/open_tee_sock`，可通过环境变量 `OPENTEE_SOCKET_FILE_PATH` 修改
2. **TA 路径**：通过 `/etc/opentee.conf` 配置，`ta_dir_path` 指向 TA .so 文件所在目录
3. **权限**：tee_manager 通常需要 root 权限创建 socket
4. **共享内存**：CA 和 TA 之间通过 memfd/shm 传递大块数据，非 socket 直接传输

---

*文档生成时间：2026-04-01*  
*基于 Open-TEE 源码分析*
