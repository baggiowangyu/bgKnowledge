# PPAPI框架简要介绍

| 版本 |     作者     | 备注 |
|:----:|:------------:|:---:|
| v1.0 | baggiowangyu |  无 |

## PPAPI组件介绍

在Chrome中，PPAPI组件可分为两种：
- 可信组件：可信的PPAPI组件可以通过平台动态库的形式（Windows下为dll文件，Linux下是so文件）由浏览器直接加载。可信的PPAPI组件以平台动态库的形式存在，所以一般Chrome沙箱内允许的API（例如CreateThread）都可以调用。

      比如：chrome --register-pepper-plugins="D:\\ppapi\\ppapi.dll;application/x-ppapi-ppapi" "http://localhost/ppapi.html"
      上面这种方式启动加载组件常用于调试使用。

- 不可信组件：不可信的PPAPI组件需要使用Native Client机制发布（以Native Client的形式发布到Chrome Web Store或者以Portable Native Client的形式发布了网址的Web服务器上），而Native Client需要维系跨平台可移植性，完全与系统API告别了，只能使用PPAPI提供的开发库和C/C++运行库。

## 一段PPAPI与Javascript交互的经典场景

```
<object id="plugin" type="application/x-ppapi-post-message-example"  width="1" height="1"/>
  function HandleMessage(message_event) {
    if (message_event.data) {
      alert("The string was a palindrome.");
    } else {
      alert("The string was not a palindrome.");
    }
  }

  function AddListener() {
    var plugin = document.getElementById("plugin");
    plugin.addEventListener("message", HandleMessage, false);
  }
  document.addEventListener("DOMContentLoaded", AddListener, false);

  function SendString() {
    var plugin = document.getElementById("plugin");
    var inputBox = document.getElementById("inputBox");

    // Send the string to the plugin using postMessage.  This results in a call
    // to Instance::HandleMessage in C++ (or PPP_Messaging::HandleMessage in C).
    plugin.postMessage(inputBox.value);
  }
```
- Javascript通过调用插件的addEventListener()接口，来增加事件监听。同时传入"message"来监听消息事件。达到Javascript接收插件发送的消息。
- Javascript通过调用插件的postMessage()接口，向插件传入消息。

## PPAPI组件导出函数说明

无论是可信组件还是不可信组件，在PPAPI中，我们需要导出三个函数提供给浏览器调用，它们分别是：
- **```int32_t PPP_InitializeModule(PP_Module module, PPB_GetInterface get_browser_interface);```**
  - 初始化PPAPI模块，传入浏览器生成的模块句柄PP_Module；
  - get_browser_interface向组件提供了浏览器宿主的常见功能接口，例如：**```PPB_Core()```**/**```PPB_Messaging()```**/**```PPB_URLLoader()```**/**```PPB_FileIO()```** 等；
- **```void PPP_ShutdownModule(void);```**
  - 关闭模块；
- **```const void* PPP_GetInterface(const char* interface_name);```**
  - 插件向浏览器宿主提供的功能接口，例如：**```PPP_Instance()```**/**```PPP_InputEvent()```**/**```PPPMessaging()```** 等；

## 浏览器宿主向PPAPI组件提供的功能接口

浏览器宿主向PPAPI组件提供的功能接口以PPB命名开头，可以从导出函数PPP_InitializeModule传入的PPB_GetInterface参数中根据接口ID查询对应的功能接口。

常用的一些功能接口有：

| 功能说明 | 接口ID | 接口定义 | 重要成员方法说明 |
|:-:|:-:|:-:|:-|
| 核心基础接口 | PPB_CORE_INTERFACE | PPB_Core | AddRefResource、ReleaseResource：增加/减少资源的引用计数<br>GetTime、GetTimeTicks：获取当前的时间<br>CallOnMainThread、IsMainThread：主线程相关的处理 |
| 消息接口 | PPB_MESSAGING_INERFACE | PPB_Messaging | PostMessage：抛送消息（Plugin->JS，JS通过addEventListener挂接message事件）<br>RegisterMessageHandler、UnregisterMessageHandler：自定义消息处理对象，用于拦截PPP_Messaging接口的HandleMessage处理（应用场景未明） |
| HTTP请求接口 | PPB_URLLOADER_INTERFACE | PPB_URLLoader | 常用的HTTP请求操作：Open、GetResponseInfo、ReadResponseBody、Close... |
| 文件IO操作 | PPB_FILEIO_INTERFACE | PPB_FileIO | File的基本操作：Create、Open、Read、Write、Close... |
| 网络操作 | PPB_UDPSOCKET_INTERFACE<br>PPB_TCPSOCKET_INTERFACE<br>PPB_HOSTRESOLVER_INTERFACE<br>... | PPB_UDPSocket<br>PPB_TCPSocket<br>PPB_HostResolver<br>... | UDP套接字、TCP套接字、域名解析服务等常用的操作接口 |
| 音视频操作 | PPB_AUDIO_INTERFACE<br>PPB_VIDEODECODER_INTERFACE<br>PPB_VEDIOENCODER_INTERFACE<br>... | PPB_Audio<br>PPB_VideoDecoder<br>PPB_VideoEncoder<br>... | 封装音视频的编解码等操作 |

此外，还有Image（图像）、2D、3D等接口，基本能满足开发本地应用的所有功能。

## PPAPI组件向浏览器宿主提供的功能接口

PPAPI组件向浏览器宿主提供个功能接口以PPP命名开头。

| 功能说明 | 接口ID | 接口定义 | 重要成员方法说明 |
|:-:|:-:|:-:|:-|
| 插件实例对象 | PPP_INSTANCE_INTERFACE | PPP_Instance | DidCreate、DidDestroy：插件创建销毁通知<br>DidChangeView、DidChangeFocus：插件尺寸、焦点改变通知<br>HandleDocumentLoad：Document加载完成通知<br>这些调用最终都映射到Instance实例的相关成员方法上。 |
| 输入接口 | PPP_INPUT_EVENT_INTRFACE | PPP_InputEvent | HandleInputEvent：处理输入消息，最终转接到Instance的HandleInputEvent接口。 |
| 消息接口 | PPP_MESSAGING_INTERFACE | PPP_Messing | HandleMessage：处理消息（JS->Plugin， JS调用插件的postMessage方法），最终映射到Instance实例的HandleMessage方法。 |

除了这三个最基础的接口外，Module对象的AddPluginInterface提供了扩展组件接口的功能，例如:

- 封装PDF插件功能的PPP_Pdf接口；
- 封装视频采集功能的PPP_VideoCapture_Dev接口；
- 封装视频硬件解码的PPP_VideoDecoder_Dev接口；
- 封装3D图形相关通知事件的PPP_Graphics3D接口；

此外还有PPP_MouseLock、PPP_Find_Private、PPP_Instance_Private等私有接口。

## 关于PPAPI组件的一些特征说明

- 插件实例：每一个插件实例都对应一个PP_Instance句柄。每一类插件都支持创建多个插件实例（浏览器打开多个带有插件元素的页面）。

- 资源：















































-
