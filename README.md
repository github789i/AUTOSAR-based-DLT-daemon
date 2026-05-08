# AUTOSAR-based-DLT-daemon

# 项目介绍

**DLT-daemon** 不仅仅是一个“写日志”的工具，它是车载 Linux 系统中不可或缺的通信中枢。

![1773564596835](DLT日志系统移植与开发.assets/1773564596835.png)

- **DLT User** 
  - 本质上是一个服务于其各自（与 DLT 无关）目的并生成 DLT 日志消息的应用程序。它利用 DLT 库来制作和传输这些消息。
- **DLT Library**
  - 为 DLT 用户（即应用程序）提供方便的 API，以创建 DLT 日志消息并将其移交给 DLT 守护进程。如果后者不可用，库会将消息缓存在环形缓冲区中，这样它们就不会立即丢失。
- **DLT Daemon** 
  - 是 ECU 的 DLT 通信接口。它收集并缓冲来自 ECU 上运行的一个或多个 DLT 用户的日志消息，并根据 DLT 客户端的请求将其提供给他们。守护进程还接受来自客户端的控制消息以调整守护进程或应用程序的行为。
- **DLT Client** 
  - 通过从 DLT 守护进程获取来自 DLT 用户的日志消息来接收和使用它们。它还可以发出控制消息来控制 DLT 守护程序或其连接的 DLT 用户的行为。 DLT 客户端甚至可以通过所谓的注入消息将用户定义的数据传输给 DLT 用户。



### 项目功能

DLT (Diagnostic Log and Trace) 的核心任务是将分布在不同进程（甚至不同处理器）中的日志，高效、标准化地汇聚并发送到外部调试终端。

- **多源日志汇聚：** 收集来自多个应用程序（Dlt User Apps）的日志和控制信息。
- **标准化格式：** 遵循 AUTOSAR 规范，将日志封装为包含 Header（时间戳、ECU ID、会话 ID）和 Payload 的二进制包。
  - **AUTOSAR**（AUTomotive Open System ARchitecture，汽车开放系统架构） 是由全球各大汽车制造商（如宝马、大众、丰田）和供应商（如博世、大陆）共同制定的一套**汽车电子软件标准** 
  - **核心理念：** **软硬件解耦**。让开发算法的人不用管芯片是哪家的，让做硬件的人只需要提供标准接口
  - AUTOSAR 的层级结构 从下到上分为三层 
    -  **微控制器抽象层 (MCAL)：** 最底层。直接操作硬件寄存器，把芯片的差异常规化。
    - **基础软件层 (BSW)：** 中间层。提供各种服务，比如网络通信（CAN/Ethernet）、内存管理、诊断服务，以及你正在看的 **DLT（Diagnostic Log and Trace）**。
    - **运行环境 (RTE)：** “隔离层”。确保应用层代码不直接接触底层，通过虚拟函数进行通信。
    - **应用软件层 (ASW)：** 最顶层。比如自动驾驶算法、空调控制逻辑等
  - Classic Platform vs. Adaptive Platform 
    - **Classic AUTOSAR (CP):**
      - **运行环境：** 运行在嵌入式 RTOS（如 OSEK）上，通常是单片机（MCU）。
      - **特点：** 实时性极强（毫秒级响应）、高度安全、静态配置。
      - **应用：** 刹车控制、引擎控制、安全气囊。
    - **Adaptive AUTOSAR (AP):**
      - **运行环境：** 运行在 **POSIX 标准的系统**（如 **Linux** 或 QNX）上。
      - **特点：** 面向服务 (SOA)、支持远程更新 (OTA)、算力强大。
      - **应用：** 智能座舱、自动驾驶、车联网
      - **DLT-daemon** 就是 **Adaptive AUTOSAR** 中用于日志记录的标准中间件实现 
  - 为什么 DLT 属于 AUTOSAR？
    - 在汽车开发中，调试非常困难。几十个 ECU（电子控制单元）通过以太网连接，如果某个功能报错，工程师需要一种**统一的格式**来查看所有 ECU 的运行状态
    - AUTOSAR 规定了 DLT 协议：
      - 所有的日志必须有统一的 Header（包含 ID、时间戳、优先级）。
      - 所有的日志必须能通过 DLT 协议导出到外部设备。
      - 这就是为什么 `dlt-daemon` 严格遵守这些规范
  - AUTOSAR 就是汽车界的“标准化语言”。
    - **学习 DLT-daemon** = 掌握了 Adaptive AUTOSAR 中的日志标准。
    - **掌握 AUTOSAR** = 拿到了进入一线车企或 Tier 1 供应商（如华为、经纬恒润、博世）的敲门砖
- **运行时控制：** 支持在不重启程序的情况下，远程动态修改日志级别（如从 INFO 改为 DEBUG）。
- **多种后端传输：** 支持通过 Serial (RS232)、TCP/IP、UDP 或存储到本地文件。
- **非侵入式缓冲：** 当后端连接断开时，日志会暂存在环形缓冲区中，防止数据丢失或阻塞应用。

# 快速复现

## 第一步：安装依赖

```Bash
sudo apt-get update
sudo apt-get install cmake zlib1g-dev libdbus-1-dev g++
```



## 第二步：编译源码

```Bash
# 重新创建并进入工作目录
mkdir -p /home/book/dlt_workspace
cd /home/book/dlt_workspace

# 克隆源码（建议克隆稳定分支）
git clone https://github.com/COVESA/dlt-daemon.git
cd dlt-daemon

# 创建独立的构建文件夹（关键步骤！）
mkdir build
cd build


# 执行 CMake 配置
# cmake ..
# 编译出错：
#	DLT 的代码对较旧版本的 GCC 或者特定的编译检查非常敏感。
#	这次报错是因为 strlen 返回的是 size_t (64位)，而代码尝试将其存入 uint8_t (8位)，
#	触发了类型转换安全检查 (-Werror=conversion)
# 重新配置：通过 -DCMAKE_C_FLAGS 强行插入“不检查转换”和“不视警告为错误”
cmake .. -DWITH_WERROR=OFF -DCMAKE_C_FLAGS="-w"

# 注意：这一步会自动根据 dlt_user.h.in 生成 dlt_user.h
# CMake 这种构建工具的核心逻辑：保护源码目录（Source Tree）的纯净。
# 	当你执行 cmake .. 时，它是以 build 目录作为“车间”的。
#	所有生成的、编译的文件，都不会出现在你的源码目录 (include/dlt) 里
#	而是会出现在 build 目录的对应位置：/dlt_workspace/dlt-daemon/build/include/dlt/
# 标准的“出厂”流程
#	虽然文件现在在 build 里，但我们并不直接从 build 里引用它。标准的做法是：
#		编译 (make)：把源码编译成二进制。
#		安装 (sudo make install)：这一步才是关键！
#		make install 会把：源码目录里的 dlt.h以及 build 目录里生成的 dlt_user.h
#		统一 拷贝到系统的标准位置：/usr/local/include/dlt/


# 编译
# 使用 -j4 加快速度（取决于你的 CPU 核心数）
make -j4


# 编译成功后的最后三步
# 当 make 跑到 100% 后，请务必执行：
# 1. 安装到系统目录
sudo make install
# 2. 刷新动态库缓存（非常重要，否则运行程序会报找不到 libdlt.so）
sudo ldconfig
# 3. 最终验证：查看头文件是否真的“各就各位”了
ls -l /usr/local/include/dlt/dlt_user.h

```



## 第三步：启动测试

现在系统里已经有了标准安装的 DLT 环境。让我们趁热打铁，完成最后一步：**编写并运行一个真正的 HelloWorld 客户端** 



### **编写代码：`hello_dlt.c`** 

- 保持目录整洁，建议把自己的代码放在 `dlt-daemon` 源码目录之外 
- cd /home/book/dlt_workspace 
-  gedit hello_dlt.c

- ```c
  // hello_dlt.c
  #include <dlt.h>
  
  DLT_DECLARE_CONTEXT(hello_ctx);
  
  int main() {
      // 注册应用
      DLT_REGISTER_APP("HELO", "Standard Hello App");
      // 注册上下文
      DLT_REGISTER_CONTEXT(hello_ctx, "TEST", "Test Context");
  
      // 发送一条带变量的日志（模拟数据）
      int count = 0;
      while(count < 5) {
          DLT_LOG(hello_ctx, DLT_LOG_INFO, DLT_STRING("Success! Count:"), DLT_INT(count));
          printf("Sending DLT message %d...\n", count);
          count++;
          sleep(1); 
      }
  
      DLT_UNREGISTER_CONTEXT(hello_ctx);
      DLT_UNREGISTER_APP();
      return 0;
  }
  ```

  

### **gcc编译**

- bash：gcc hello_dlt.c -o hello_dlt -ldlt

- ```bash
  # 报错
  hello_dlt.c:1:10: fatal error: dlt.h: No such file or directory
   #include <dlt.h>
            ^~~~~~~
  compilation terminated.
  
  # 原因
  # 因为在标准安装下，DLT 的头文件通常被放置在 /usr/local/include/dlt/ 这个子目录里
  # 虽然文件在系统路径下，但 GCC 默认只会搜索 /usr/local/include 根目录
  # 它不会自动钻进 dlt/ 文件夹里去找
  
  # 解决办法
  # 方法 A：修改编译命令（推荐，不改代码）
  # 通过 -I 参数显式告诉编译器头文件所在的具体文件夹
  gcc hello_dlt.c -o hello_dlt -I /usr/local/include/dlt -ldlt
  ```
