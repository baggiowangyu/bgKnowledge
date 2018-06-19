# GmSSL的编译方法

## 什么是GmSSL

```
GmSSL是一个开源的密码工具箱，支持SM2/SM3/SM4/SM9等国密(国家商用密码)算法、SM2国密数字证书及基于SM2证书的SSL/TLS安全通信协议，支持国密硬件密码设备，提供符合国密规范的编程接口与命令行工具，可以用于构建PKI/CA、安全通信、数据加密等符合国密标准的安全应用。
GmSSL项目是OpenSSL项目的分支，并与OpenSSL保持接口兼容。因此GmSSL可以替代应用中的OpenSSL组件，并使应用自动具备基于国密的安全能力。
GmSSL项目采用对商业应用友好的类BSD开源许可证，开源且可以用于闭源的商业应用。
GmSSL项目由北京大学关志副研究员的密码学研究组开发维护，项目源码托管于GitHub。

自2014年发布以来，GmSSL已经在多个项目和产品中获得部署与应用，并获得2015年度“一铭杯”中国Linux软件大赛二等奖(年度最高奖项)与开源中国密码类推荐项目。
GmSSL项目的核心目标是通过开源的密码技术推动国内网络空间安全建设。
```

## 在Windows环境下编译和安装

安装ActivePerl和Visual Studio，以管理员身份打开Visual Studio Tools下的Developer Command Prompt控制台并运行：

```
perl Configure VC-WIN32
nmake
nmake install
```

下面是一些常见的编译错误及原因：

- 安装最新的Visual Studio版本，不要使用 **```Visual C++ 6```**、**```Visual Studio 2008```**。
- 确保从一份干净的（没有已经编译出来的对象文件或汇编文件）最新Master分支源代码开始编译。
- 编译系统没有找到nmake。实际上nmake是Visual Studio自带的工具，不需要单独安装。编译系统无法找到nmake的原因是没有在Visual Studio的命令行环境下执行编译指定。
- 无法执行nmake install。这个命令需要以管理员的身份执行。
对象文件(.obj)和目标平台不一致，通常是由于在Visual Studio的32位控制台下执行perl Configure VC-WIN64A；或者在Visual Studio的64位控制台下执行perl Configure VC-WIN32导致的。
