# 基于Live555的历史视频点播服务视频兼容服务

| 版本号 | 作者 | 修改日期 | 说明 |
|:--:|:--:|:--:|:--:|
| v1.0 | **```wangy```** | 2018-06-20 | 基于原生Live555开发的历史视音频点播服务 |

## 技术背景说明

- 行业内普遍采用执法记录仪等单兵摄录设备对执法行为进行摄录取证；
- 作为执法证据，视频文件本身不应被修改；
- 在本系统中，需要对视音频文件调用播放；
- 在平台级联时，能够以GB/T 28181标准向上级平台推送视频数据；
- 执法仪行业没有统一标准，摄录保存的文件格式多种多样；

## 需要解决的技术边界

- 执法仪录制的各类视音频文件能够被远程播放调用；
- 4G执法仪实时回传录制的视频能够被远程播放调用；
- 远程播放时，应能支持播放、暂停、停止、快放、慢放、跳转播放等功能；

## 关于视音频复用格式与视音频编码的支撑对照表

| 视音频复用格式 | 视频编码 | 音频编码 | 视音频来源 | 原生Live555测试结果 |
|:--:|:--:|:--:|:--:|:--:|
| MP4 | H264(AVC) | AAC | 执法仪 | 不兼容 |
| MP4 | H264(AVC) | MP3 | 执法仪 | 不兼容 |
| MP4 | H264(MPEG4) | AAC | 执法仪 | 不兼容 |
| MP4 | H264(MPEG4) | MP3 | 执法仪 | 不兼容 |
| MP4 | H265(HEVC) | AAC | 执法仪 | 不兼容 |
| MP4 | H265(HEVC) | MP3 | 执法仪 | 不兼容 |
| AVI | H264(AVC) | U-Law | 执法仪 | 不兼容 |
| AVI | MJPG | PCM | 执法仪 | 不兼容 |
| WMV | WMV3 | WMA | 执法仪 | 不兼容 |
| GMFv1 | H264 | G711u | 4G实时回传录制 | 不兼容 |
| GMFv2 | H264 | G711u | 4G实时回传录制 | 不兼容 |

## 技术实现原理说明

### 主程序初始化

#### 1.服务启动

首先Live555会初始化我们的运行环境，如下代码：

```
  TaskScheduler* scheduler = BasicTaskScheduler::createNew();
  UsageEnvironment* env = BasicUsageEnvironment::createNew(*scheduler);
```

如果需要给RTSP服务增加接入控制的话，需要激活 **```ACCESS_CONTROL```** 宏：

```
  UserAuthenticationDatabase* authDB = NULL;
#ifdef ACCESS_CONTROL
  // To implement client access control to the RTSP server, do the following:
  authDB = new UserAuthenticationDatabase;
  authDB->addUserRecord("username1", "password1"); // replace these with real strings
  // Repeat the above with each <username>, <password> that you wish to allow
  // access to the server.
#endif
```

接下来创建RTSP服务对象。首先尝试默认端口554，如果554被占用的话则尝试使用备用端口8554：

```
  // Create the RTSP server.  Try first with the default port number (554),
  // and then with the alternative port number (8554):
  RTSPServer* rtspServer;
  portNumBits rtspServerPortNum = 554;
  rtspServer = DynamicRTSPServer::createNew(*env, rtspServerPortNum, authDB);
  if (rtspServer == NULL) {
    rtspServerPortNum = 8554;
    rtspServer = DynamicRTSPServer::createNew(*env, rtspServerPortNum, authDB);
  }
  if (rtspServer == NULL) {
    *env << "Failed to create RTSP server: " << env->getResultMsg() << "\n";
    exit(1);
  }

```

接下来获取可以访问的URL地址前缀，一般是以 **```rtsp://127.0.0.1/```** 的形式展现的

```
  char* urlPrefix = rtspServer->rtspURLPrefix();
```

接下来开启RTSP-over-HTTP通道，默认开启80、8000、8080，如果有任意一个端口开启成功了，就能通过http访问视频：

```
  // Also, attempt to create a HTTP server for RTSP-over-HTTP tunneling.
  // Try first with the default HTTP port (80), and then with the alternative HTTP
  // port numbers (8000 and 8080).

  if (rtspServer->setUpTunnelingOverHTTP(80) || rtspServer->setUpTunnelingOverHTTP(8000) || rtspServer->setUpTunnelingOverHTTP(8080)) {
    *env << "(We use port " << rtspServer->httpServerPortNum() << " for optional RTSP-over-HTTP tunneling, or for HTTP live streaming (for indexed Transport Stream files only).)\n";
  } else {
    *env << "(RTSP-over-HTTP tunneling is not available.)\n";
  }
```

最后，就进入了事件循环，等待RTSP请求并处理：

```
  env->taskScheduler().doEventLoop(); // does not return
```

#### 2.doEventLoop()逻辑分析

实际上，doEventLoop()函数里面是一个死循环，只执行了一个函数SingleStep()：

```
void BasicTaskScheduler0::doEventLoop(char volatile* watchVariable) {
  // Repeatedly loop, handling readble sockets and timed events:
  while (1) {
    if (watchVariable != NULL && *watchVariable != 0) break;
    SingleStep();
  }
}
```

#### 3. SingleStep()逻辑分析

首先调用select()等待网络消息输入：

```
BasicTaskScheduler.cpp(66):

	fd_set readSet = fReadSet; // make a copy for this select() call
	fd_set writeSet = fWriteSet; // ditto
	fd_set exceptionSet = fExceptionSet; // ditto

	DelayInterval const& timeToDelay = fDelayQueue.timeToNextAlarm();
	struct timeval tv_timeToDelay;
	tv_timeToDelay.tv_sec = timeToDelay.seconds();
	tv_timeToDelay.tv_usec = timeToDelay.useconds();
	// Very large "tv_sec" values cause select() to fail.
	// Don't make it any larger than 1 million seconds (11.5 days)
	const long MAX_TV_SEC = MILLION;
	if (tv_timeToDelay.tv_sec > MAX_TV_SEC) {
		tv_timeToDelay.tv_sec = MAX_TV_SEC;
	}

	// Also check our "maxDelayTime" parameter (if it's > 0):
	if (maxDelayTime > 0 &&
		(tv_timeToDelay.tv_sec > (long)maxDelayTime/MILLION ||
		(tv_timeToDelay.tv_sec == (long)maxDelayTime/MILLION &&
		tv_timeToDelay.tv_usec > (long)maxDelayTime%MILLION)))
	{
			tv_timeToDelay.tv_sec = maxDelayTime/MILLION;
			tv_timeToDelay.tv_usec = maxDelayTime%MILLION;
	}

	// 这里监视网络消息
	int selectResult = select(fMaxNumSockets, &readSet, &writeSet, &exceptionSet, &tv_timeToDelay);
	if (selectResult < 0)
	{
#if defined(__WIN32__) || defined(_WIN32)
		int err = WSAGetLastError();
		// For some unknown reason, select() in Windoze sometimes fails with WSAEINVAL if
		// it was called with no entries set in "readSet".  If this happens, ignore it:
		if (err == WSAEINVAL && readSet.fd_count == 0) {
			err = EINTR;
			// To stop this from happening again, create a dummy socket:
			if (fDummySocketNum >= 0) closeSocket(fDummySocketNum);
			fDummySocketNum = socket(AF_INET, SOCK_DGRAM, 0);
			FD_SET((unsigned)fDummySocketNum, &fReadSet);
		}
		if (err != EINTR)
		{
#else
		if (errno != EINTR && errno != EAGAIN) {
#endif
			// Unexpected error - treat this as fatal:
#if !defined(_WIN32_WCE)
			perror("BasicTaskScheduler::SingleStep(): select() fails");
			// Because this failure is often "Bad file descriptor" - which is caused by an invalid socket number (i.e., a socket number
			// that had already been closed) being used in "select()" - we print out the sockets that were being used in "select()",
			// to assist in debugging:
			fprintf(stderr, "socket numbers used in the select() call:");
			for (int i = 0; i < 10000; ++i)
			{
				if (FD_ISSET(i, &fReadSet) || FD_ISSET(i, &fWriteSet) || FD_ISSET(i, &fExceptionSet))
				{
					fprintf(stderr, " %d(", i);
					if (FD_ISSET(i, &fReadSet)) fprintf(stderr, "r");
					if (FD_ISSET(i, &fWriteSet)) fprintf(stderr, "w");
					if (FD_ISSET(i, &fExceptionSet)) fprintf(stderr, "e");
					fprintf(stderr, ")");
				}
			}
			fprintf(stderr, "\n");
#endif
			internalError();
		}
	}
```

### RTSP网络消息入口

#### 1.RTSP请求消息入口

关注GenericMediaServer.cpp(237)

```
void GenericMediaServer::ClientConnection::incomingRequestHandler(void* instance, int /*mask*/) {
  ClientConnection* connection = (ClientConnection*)instance;
  connection->incomingRequestHandler();
}
```

这里是输入请求的地方

### 本地视频文件分析入口

### 1.

ServerMediaSession* DynamicRTSPServer::lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession) 是媒体文件分析的入口

```
ServerMediaSession* DynamicRTSPServer
	::lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession) {
		// First, check wheth er the specified "streamName" exists as a local file:
    // 首先，确认传入的streamName是否是本地存在的一个文件
		FILE* fid = fopen(streamName, "rb");
		Boolean fileExists = fid != NULL;

		// Next, check whether we already have a "ServerMediaSession" for this file:
    // 然后，检查这个文件的ServerMediaSession会话是否已经存在
		ServerMediaSession* sms = RTSPServer::lookupServerMediaSession(streamName);
		Boolean smsExists = sms != NULL;

		// Handle the four possibilities for "fileExists" and "smsExists":
    // 接下来对fileExists和smsExists两个变量存在的四种可能性进行处理
		if (!fileExists) {
			if (smsExists) {
				// "sms" was created for a file that no longer exists. Remove it:
        // 文件不存在，但是对应的 sms 存在，说明这个sms已经不再使用了，移除
				removeServerMediaSession(sms);
				sms = NULL;
			}

			return NULL;
		} else {
			if (smsExists && isFirstLookupInSession) {
				// Remove the existing "ServerMediaSession" and create a new one, in case the underlying
				// file has changed in some way:
        // 移除已经存在的 ServerMediaSession 并创建一个新的，防止底层文件通过某种手段被修改了
				removeServerMediaSession(sms);
				sms = NULL;
			}

			if (sms == NULL) {
        // 这里很重要，在这个地方其实已经做了解复用，然后将视频流、音频流、文本流 作为Session做成链表组织起来
				// 后面进行处理发流
				sms = createNewSMS(envir(), streamName, fid);
				addServerMediaSession(sms);
			}

			fclose(fid);
			return sms;
		}
}
```

实际上，```createNewSMS()```函数是核心的解复用操作，需要详细剖析

```
static ServerMediaSession* createNewSMS(UsageEnvironment& env,
	char const* fileName, FILE* /*fid*/) {
		// Use the file name extension to determine the type of "ServerMediaSession":

		char const* extension = strrchr(fileName, '.');
		if (extension == NULL) return NULL;

		ServerMediaSession* sms = NULL;
		Boolean const reuseSource = False;
		if (strcmp(extension, ".aac") == 0) {
			// Assumed to be an AAC Audio (ADTS format) file:
			NEW_SMS("AAC Audio");
			sms->addSubsession(ADTSAudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".amr") == 0) {
			// Assumed to be an AMR Audio file:
			NEW_SMS("AMR Audio");
			sms->addSubsession(AMRAudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".ac3") == 0) {
			// Assumed to be an AC-3 Audio file:
			NEW_SMS("AC-3 Audio");
			sms->addSubsession(AC3AudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".m4e") == 0) {
			// Assumed to be a MPEG-4 Video Elementary Stream file:
			NEW_SMS("MPEG-4 Video");
			sms->addSubsession(MPEG4VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".264") == 0) {
			// Assumed to be a H.264 Video Elementary Stream file:
			NEW_SMS("H.264 Video");
			OutPacketBuffer::maxSize = 100000; // allow for some possibly large H.264 frames
			sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".265") == 0) {
			// Assumed to be a H.265 Video Elementary Stream file:
			NEW_SMS("H.265 Video");
			OutPacketBuffer::maxSize = 100000; // allow for some possibly large H.265 frames
			sms->addSubsession(H265VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".mp3") == 0) {
			// Assumed to be a MPEG-1 or 2 Audio file:
			NEW_SMS("MPEG-1 or 2 Audio");
			// To stream using 'ADUs' rather than raw MP3 frames, uncomment the following:
			//#define STREAM_USING_ADUS 1
			// To also reorder ADUs before streaming, uncomment the following:
			//#define INTERLEAVE_ADUS 1
			// (For more information about ADUs and interleaving,
			//  see <http://www.live555.com/rtp-mp3/>)
			Boolean useADUs = False;
			Interleaving* interleaving = NULL;
#ifdef STREAM_USING_ADUS
			useADUs = True;
#ifdef INTERLEAVE_ADUS
			unsigned char interleaveCycle[] = {0,2,1,3}; // or choose your own...
			unsigned const interleaveCycleSize
				= (sizeof interleaveCycle)/(sizeof (unsigned char));
			interleaving = new Interleaving(interleaveCycleSize, interleaveCycle);
#endif
#endif
			sms->addSubsession(MP3AudioFileServerMediaSubsession::createNew(env, fileName, reuseSource, useADUs, interleaving));
		} else if (strcmp(extension, ".mpg") == 0) {
			// Assumed to be a MPEG-1 or 2 Program Stream (audio+video) file:
			NEW_SMS("MPEG-1 or 2 Program Stream");
			MPEG1or2FileServerDemux* demux
				= MPEG1or2FileServerDemux::createNew(env, fileName, reuseSource);
			sms->addSubsession(demux->newVideoServerMediaSubsession());
			sms->addSubsession(demux->newAudioServerMediaSubsession());
		} else if (strcmp(extension, ".vob") == 0) {
			// Assumed to be a VOB (MPEG-2 Program Stream, with AC-3 audio) file:
			NEW_SMS("VOB (MPEG-2 video with AC-3 audio)");
			MPEG1or2FileServerDemux* demux
				= MPEG1or2FileServerDemux::createNew(env, fileName, reuseSource);
			sms->addSubsession(demux->newVideoServerMediaSubsession());
			sms->addSubsession(demux->newAC3AudioServerMediaSubsession());
		} else if (strcmp(extension, ".ts") == 0) {
			// Assumed to be a MPEG Transport Stream file:
			// Use an index file name that's the same as the TS file name, except with ".tsx":
			unsigned indexFileNameLen = strlen(fileName) + 2; // allow for trailing "x\0"
			char* indexFileName = new char[indexFileNameLen];
			sprintf(indexFileName, "%sx", fileName);
			NEW_SMS("MPEG Transport Stream");
			sms->addSubsession(MPEG2TransportFileServerMediaSubsession::createNew(env, fileName, indexFileName, reuseSource));
			delete[] indexFileName;
		} else if (strcmp(extension, ".wav") == 0) {
			// Assumed to be a WAV Audio file:
			NEW_SMS("WAV Audio Stream");
			// To convert 16-bit PCM data to 8-bit u-law, prior to streaming,
			// change the following to True:
			Boolean convertToULaw = False;
			sms->addSubsession(WAVAudioFileServerMediaSubsession::createNew(env, fileName, reuseSource, convertToULaw));
		} else if (strcmp(extension, ".dv") == 0) {
			// Assumed to be a DV Video file
			// First, make sure that the RTPSinks' buffers will be large enough to handle the huge size of DV frames (as big as 288000).
			OutPacketBuffer::maxSize = 300000;

			NEW_SMS("DV Video");
			sms->addSubsession(DVVideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
		} else if (strcmp(extension, ".mkv") == 0 || strcmp(extension, ".webm") == 0) {
			// Assumed to be a Matroska file (note that WebM ('.webm') files are also Matroska files)
			OutPacketBuffer::maxSize = 100000; // allow for some possibly large VP8 or VP9 frames
			NEW_SMS("Matroska video+audio+(optional)subtitles");

			// Create a Matroska file server demultiplexor for the specified file.
			// (We enter the event loop to wait for this to complete.)
			MatroskaDemuxCreationState creationState;
			creationState.watchVariable = 0;
			MatroskaFileServerDemux::createNew(env, fileName, onMatroskaDemuxCreation, &creationState);
			env.taskScheduler().doEventLoop(&creationState.watchVariable);

			ServerMediaSubsession* smss;
			while ((smss = creationState.demux->newServerMediaSubsession()) != NULL) {
				sms->addSubsession(smss);
			}
		} else if (strcmp(extension, ".ogg") == 0 || strcmp(extension, ".ogv") == 0 || strcmp(extension, ".opus") == 0) {
			// Assumed to be an Ogg file
			NEW_SMS("Ogg video and/or audio");

			// Create a Ogg file server demultiplexor for the specified file.
			// (We enter the event loop to wait for this to complete.)
			OggDemuxCreationState creationState;
			creationState.watchVariable = 0;
			OggFileServerDemux::createNew(env, fileName, onOggDemuxCreation, &creationState);
			env.taskScheduler().doEventLoop(&creationState.watchVariable);

			ServerMediaSubsession* smss;
			while ((smss = creationState.demux->newServerMediaSubsession()) != NULL) {
				sms->addSubsession(smss);
			}
		}

		return sms;
}
```

在整个创建过程中，有一宏非常重要，它就是：```NEW_SMS```

```
#define NEW_SMS(description) do {\
	char const* descStr = description\
	", streamed by the LIVE555 Media Server";\
	sms = ServerMediaSession::createNew(env, fileName, fileName, descStr);\
} while(0)
```

### 2. 视音频的解复用

实际上在RTSP协议中的DESCRIBE命令的时候就应该进行视音频的解复用了，具体代码如下：

```
void RTSPServer::RTSPClientConnection
::handleCmd_DESCRIBE(char const* urlPreSuffix, char const* urlSuffix, char const* fullRequestStr) {
  ServerMediaSession* session = NULL;
  char* sdpDescription = NULL;
  char* rtspURL = NULL;
  do {
    char urlTotalSuffix[2*RTSP_PARAM_STRING_MAX];
        // enough space for urlPreSuffix/urlSuffix'\0'
    urlTotalSuffix[0] = '\0';
    if (urlPreSuffix[0] != '\0') {
      strcat(urlTotalSuffix, urlPreSuffix);
      strcat(urlTotalSuffix, "/");
    }
    strcat(urlTotalSuffix, urlSuffix);

    if (!authenticationOK("DESCRIBE", urlTotalSuffix, fullRequestStr)) break;

    // We should really check that the request contains an "Accept:" #####
    // for "application/sdp", because that's what we're sending back #####

    // Begin by looking up the "ServerMediaSession" object for the specified "urlTotalSuffix":
    session = fOurServer.lookupServerMediaSession(urlTotalSuffix);
    if (session == NULL) {
      handleCmd_notFound();
      break;
    }

    // Increment the "ServerMediaSession" object's reference count, in case someone removes it
    // while we're using it:
    session->incrementReferenceCount();

    // Then, assemble a SDP description for this session:
    // 实际上，解复用
    sdpDescription = session->generateSDPDescription();
    if (sdpDescription == NULL) {
      // This usually means that a file name that was specified for a
      // "ServerMediaSubsession" does not exist.
      setRTSPResponse("404 File Not Found, Or In Incorrect Format");
      break;
    }
    unsigned sdpDescriptionSize = strlen(sdpDescription);

    // Also, generate our RTSP URL, for the "Content-Base:" header
    // (which is necessary to ensure that the correct URL gets used in subsequent "SETUP" requests).
    rtspURL = fOurRTSPServer.rtspURL(session, fClientInputSocket);

    snprintf((char*)fResponseBuffer, sizeof fResponseBuffer,
	     "RTSP/1.0 200 OK\r\nCSeq: %s\r\n"
	     "%s"
	     "Content-Base: %s/\r\n"
	     "Content-Type: application/sdp\r\n"
	     "Content-Length: %d\r\n\r\n"
	     "%s",
	     fCurrentCSeq,
	     dateHeader(),
	     rtspURL,
	     sdpDescriptionSize,
	     sdpDescription);
  } while (0);

  if (session != NULL) {
    // Decrement its reference count, now that we're done using it:
    session->decrementReferenceCount();
    if (session->referenceCount() == 0 && session->deleteWhenUnreferenced()) {
      fOurServer.removeServerMediaSession(session);
    }
  }

  delete[] sdpDescription;
  delete[] rtspURL;
}
```

这里可以观察调用堆栈的情况：

【AAC】

![](assets/1001/20180625-90735c5e.png)  


在创建RTPSink的时候，最终创建了MPEG4GenericRTPSink，这里可能需要关注看看是不是所有支持的格式都创建的是这个Sink

```
RTPSink* ADTSAudioFileServerMediaSubsession
::createNewRTPSink(Groupsock* rtpGroupsock,
		   unsigned char rtpPayloadTypeIfDynamic,
		   FramedSource* inputSource) {
  ADTSAudioFileSource* adtsSource = (ADTSAudioFileSource*)inputSource;
  return MPEG4GenericRTPSink::createNew(envir(), rtpGroupsock,
					rtpPayloadTypeIfDynamic,
					adtsSource->samplingFrequency(),
					"audio", "AAC-hbr", adtsSource->configStr(),
					adtsSource->numChannels());
}
```
