# FFmpeg实现音频转码

| 版本 |     作者     | 日期 | 备注 |
|:----:|:------------:|:---:|:---:|
| v1.0 | baggiowangyu | 2018-06-06 | 无 |

## FFmpeg音频转码流程介绍

参考文档：http://ffmpeg.org/doxygen/trunk/group__lavc__encdec.html

在FFmpeg 3之后的版本中，提供了更加方便的接口进行编解码操作。
avocec_send_packet() / avcodec_recevice_frame() / avcodec_send_frame() / avcodec_recevice_packet() 函数提供了对应的编解码功能，并将输入输出分离开来。

这套接口可以非常方便的实现视音频的编解码操作，具体步骤如下：
1. 初始化AVcodecContext结构，并打开；
2. 将有效数据输入：
  - 对于解码来说，调用 **```avcodec_send_packet()```** 函数给解码器送入 **```AVPacket```** 中的原始压缩数据。
  - 对于编码来说，调用 **```avcodec_send_frame()```** 


-
