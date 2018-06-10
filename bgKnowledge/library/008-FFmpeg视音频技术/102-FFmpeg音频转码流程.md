# FFmpeg音频转码流程

## 输入文件的预处理

- 目标：得到处理后的 **```AVFormatContext```** 和 **```AVCodecContext```** 结构；
- 具体工作：
  - 解复用，填充好 **```AVFormatContext```** 结构数据；
  - 查找音频解码器；
  - 申请 **```AVCodecContext```** 解码器上下文结构内存；
  - 在解码器上下文填充参数；
  - 打开解码器；

## 输出文件的预处理

- 目标：根据输出文件名和输入文件的解码器上下文，得到处理后的输出文件 **```AVFormatContext```** 和 **```AVCodecContext```** 结构
- 具体工作：
  - 打开输出文件；
  - 申请 **```AVFormatContext```** 结构数据，并填充pb成员变量；
  - 通过

## 重采样的预处理
