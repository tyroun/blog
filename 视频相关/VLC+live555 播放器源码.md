{% raw %}

# VLC+live555 播放器源码

## VLC 调用流程

modules/access/live555.cpp

### 1 Open

```c
static int  Open ( vlc_object_t *p_this ){
    //初始化vlc demux的回调
    p_demux->pf_demux  = Demux;
    p_demux->pf_control= Control;
    p_demux->p_sys     = p_sys = (demux_sys_t*)calloc( 1, sizeof( demux_sys_t ) );

    //创建live555用的基本的调度器，最终这个调度器来实现select然后回调
    if( ( p_sys->scheduler = BasicTaskScheduler::createNew() ) == NULL )
    {
        msg_Err( p_demux, "BasicTaskScheduler::createNew failed" );
        goto error;
    }

    //打开sdp
    if( ( i_return = SessionsSetup( p_demux ) ) != VLC_SUCCESS )
    {
        msg_Err( p_demux, "Nothing to play for %s", p_sys->psz_pl_url );
        goto error;
    }

    //播放
    if( ( i_return = Play( p_demux ) ) != VLC_SUCCESS )
            goto error;
}
```

### 2 SessionSetup

```c
static int SessionsSetup( demux_t *p_demux ){
    //...
    //创建MediaSession
    if(!( p_sys->ms = MediaSession::createNew( *p_sys->env, p_sys->p_sdp )))
    {
        msg_Err( p_demux, "Could not create the RTSP Session: %s",
            p_sys->env->getResultMsg() );
        return VLC_EGENERIC;
    }

    /* 每个v=xxx会创建一个subsession,遍历*/
    iter = new MediaSubsessionIterator( *p_sys->ms );
    while( ( sub = iter->next() ) != NULL )
    {
        bInit = sub->initiate( 0 );
        //确定rtp socket
        if( sub->rtpSource() != NULL )
        {
            int fd = sub->rtpSource()->RTPgs()->socketNum();

            /* Increase the buffer size */
            if( i_receive_buffer > 0 )
                increaseReceiveBufferTo( *p_sys->env, fd, i_receive_buffer );

            /* Increase the RTP reorder timebuffer just a bit */
            sub->rtpSource()->setPacketReorderingThresholdTime(thresh);
        }

    }
}
```

### 3 Demux

```c
static int Demux( demux_t *p_demux )
{
    /* First warn we want to read data */
    p_sys->event_data = 0;
    for( i = 0; i < p_sys->i_track; i++ )
    {
        live_track_t *tk = p_sys->track[i];

        if( tk->waiting == 0 )
        {
            tk->waiting = 1;
            //live555 会在这里注册taskScheduler的回调 StreamRead/StreamClose
            tk->sub->readSource()->getNextFrame( tk->p_buffer, tk->i_buffer,
                                          StreamRead, tk, StreamClose, tk );
        }
    }
    /* 每300ms如果没数据，会把p_sys->event_data设为0xff，退出下面的eventloop */
    task = p_sys->scheduler->scheduleDelayedTask( 300000, TaskInterruptData, p_demux );

    /* Do the read, live555的loop */
    p_sys->scheduler->doEventLoop( &p_sys->event_data );

    /* remove the task */
    p_sys->scheduler->unscheduleDelayedTask( task );
    //这里的scheduler是BasicTaskScheduler::createNew创建的

}
```

## live555 调用流程

### 1 调度器相关

BasicTaskScheduler <- BasicTaskScheduler0 <- TaskScheduler

#### 1.1 创建函数

```c++
//maxSchedulerGranularity 默认是10000,表示10s
BasicTaskScheduler::BasicTaskScheduler(unsigned maxSchedulerGranularity)
  : fMaxSchedulerGranularity(maxSchedulerGranularity), fMaxNumSockets(0), fDummySocketNum(-1)
{
  //初始化FD_SET用于select
  FD_ZERO(&fReadSet);
  FD_ZERO(&fWriteSet);
  FD_ZERO(&fExceptionSet);

  //调用scheduleDelayedTask，会把一个超时的任务放到fDelayQueue里，select的时候会根据queue里指示的delay值，设置delay时间
  if (maxSchedulerGranularity > 0) schedulerTickTask();
}

//上面的子类调用了基类BasicTaskScheduler0的默认构造函数
BasicTaskScheduler0::BasicTaskScheduler0()
  : fLastHandledSocketNum(-1), fTriggersAwaitingHandling(0), fLastUsedTriggerMask(1), fLastUsedTriggerNum(MAX_NUM_EVENT_TRIGGERS-1) {
  //fHandlers维护了1个双向链表，用于存放socket号，还有回调。这些信息统一放在HandlerDescriptor中
  fHandlers = new HandlerSet;
  for (unsigned i = 0; i < MAX_NUM_EVENT_TRIGGERS; ++i) {
    fTriggeredEventHandlers[i] = NULL;
    fTriggeredEventClientDatas[i] = NULL;
  }
}

```

#### 1.2 回调触发

有 2 种方式

1. 是调用 BasicTaskScheduler::setBackgroundHandling，在 fHandlers 中注册回调，绑定 socket 号。在 SingleStep 时根据触发的 socket 号，调用回调
2. 调用 BasicTaskScheduler0::createEventTrigger，注册回调并且申请 EventTriggerId。调用 BasicTaskScheduler0::triggerEvent 表示该回调即将被触发，置位
   fTriggersAwaitingHandling。在 SingleStep 中，每次 select 返回后，调用 fTriggersAwaitingHandling 被置位的回调

#### 1.2 调度器启动

最终都是调用 BasicTaskScheduler::SingleStep

```c++
void BasicTaskScheduler0::doEventLoop(char volatile* watchVariable) {
  while (1) {
    if (watchVariable != NULL && *watchVariable != 0) break;
    SingleStep(); //不停调用SingleStep
  }
}


void BasicTaskScheduler::SingleStep(unsigned maxDelayTime) {
  fd_set readSet = fReadSet; // make a copy for this select() call
  fd_set writeSet = fWriteSet; // ditto
  fd_set exceptionSet = fExceptionSet; // ditto

  //...一堆为了计算超时
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
	tv_timeToDelay.tv_usec > (long)maxDelayTime%MILLION))) {
    tv_timeToDelay.tv_sec = maxDelayTime/MILLION;
    tv_timeToDelay.tv_usec = maxDelayTime%MILLION;
  }
  //...一堆为了计算超时

  int selectResult = select(fMaxNumSockets, &readSet, &writeSet, &exceptionSet, &tv_timeToDelay);
  if (selectResult < 0) {
    ...//错误处理
  }

  // 调用fHandlers回调
  HandlerIterator iter(*fHandlers);
  HandlerDescriptor* handler;
  // To ensure forward progress through the handlers, begin past the last
  // socket number that we handled:
  if (fLastHandledSocketNum >= 0) {
    while ((handler = iter.next()) != NULL) {
      if (handler->socketNum == fLastHandledSocketNum) break;
    }
    if (handler == NULL) {
      fLastHandledSocketNum = -1;
      iter.reset(); // start from the beginning instead
    }
  }
  while ((handler = iter.next()) != NULL) {
    int sock = handler->socketNum; // alias
    int resultConditionSet = 0;
    if (FD_ISSET(sock, &readSet) && FD_ISSET(sock, &fReadSet)/*sanity check*/) resultConditionSet |= SOCKET_READABLE;
    if (FD_ISSET(sock, &writeSet) && FD_ISSET(sock, &fWriteSet)/*sanity check*/) resultConditionSet |= SOCKET_WRITABLE;
    if (FD_ISSET(sock, &exceptionSet) && FD_ISSET(sock, &fExceptionSet)/*sanity check*/) resultConditionSet |= SOCKET_EXCEPTION;
    if ((resultConditionSet&handler->conditionSet) != 0 && handler->handlerProc != NULL) {
      fLastHandledSocketNum = sock;
          // Note: we set "fLastHandledSocketNum" before calling the handler,
          // in case the handler calls "doEventLoop()" reentrantly.
      (*handler->handlerProc)(handler->clientData, resultConditionSet);
      break;
    }
  }
  if (handler == NULL && fLastHandledSocketNum >= 0) {
    // We didn't call a handler, but we didn't get to check all of them,
    // so try again from the beginning:
    ...
  }

  // 调用event trigger回调
  if (fTriggersAwaitingHandling != 0) {
    if (fTriggersAwaitingHandling == fLastUsedTriggerMask) {
      // Common-case optimization for a single event trigger:
      fTriggersAwaitingHandling &=~ fLastUsedTriggerMask;
      if (fTriggeredEventHandlers[fLastUsedTriggerNum] != NULL) {
	(*fTriggeredEventHandlers[fLastUsedTriggerNum])(fTriggeredEventClientDatas[fLastUsedTriggerNum]);
      }
    } else {
      // Look for an event trigger that needs handling (making sure that we make forward progress through all possible triggers):
      unsigned i = fLastUsedTriggerNum;
      EventTriggerId mask = fLastUsedTriggerMask;

      do {
	i = (i+1)%MAX_NUM_EVENT_TRIGGERS;
	mask >>= 1;
	if (mask == 0) mask = 0x80000000;

	if ((fTriggersAwaitingHandling&mask) != 0) {
	  fTriggersAwaitingHandling &=~ mask;
	  if (fTriggeredEventHandlers[i] != NULL) {
	    (*fTriggeredEventHandlers[i])(fTriggeredEventClientDatas[i]);
	  }

	  fLastUsedTriggerMask = mask;
	  fLastUsedTriggerNum = i;
	  break;
	}
      } while (i != fLastUsedTriggerNum);
    }
  }

  // 超时回调.
  fDelayQueue.handleAlarm();
}
```

### 2 RTP 数据处理流程

#### 2.1 MediaSubsession::initiate

主要做了 2 件事，创建 SimpleRTPSource + 创建 RTCPInstance

MediaSession.cpp

```c++
Boolean MediaSubsession::initiate(int useSpecialRTPoffset) {
    //找到一对socket用于rtp和rtcp
    fRTPSocket = new Groupsock(env(), tempAddr, 0, 255);

    //收帧buffer
    increaseReceiveBufferTo(env(), fRTPSocket->socketNum(), rtpBufSize);

    //根据sdp中的a=rtpmap:96 MP2P/90000来创建不同的source
    // Create "fRTPSource" and "fReadSource":
    // MP2P是调用SimpleRTPSource
    if (!createSourceObjects(useSpecialRTPoffset)) break;

    //创建rtcp instance
    fRTCPInstance = RTCPInstance::createNew(env(), fRTCPSocket,
					      totSessionBandwidth,
					      (unsigned char const*)
					      fParent.CNAME(),
					      NULL /* we're a client */,
					      fRTPSource);

}
```

#### 2.2 MediaSubsession:: createSourceObjects 创建 source

根据 sdp，选择了 SimpleRTPSource ::createNew

SimpleRTPSource --> MultiFramedRTPSource --> RTPSource --> FramedSource --> MediaSouce --> Medium

#### 2.3 RTP 接收数据流程

vlc/Demux() -> FramedSource::getNextFrame -> MultiFramedRTPSource::doGetNextFrame

​ |-> RTPInterface::startNetworkReading 挂载回调

eventloop select 之后 -> MultiFramedRTPSource::networkReadHandler1() -> MultiFramedRTPSource::doGetNextFrame1() ->
afterGetting() -> vlc/StreamRead()

```C++
//对外的接口，用于设置读完一帧的回调。VLC在Demux中调用了这个接口
void FramedSource::getNextFrame(unsigned char* to, unsigned maxSize,
				afterGettingFunc* afterGettingFunc,
				void* afterGettingClientData,
				onCloseFunc* onCloseFunc,
				void* onCloseClientData) {

  。。。
  fAfterGettingFunc = afterGettingFunc;
  fAfterGettingClientData = afterGettingClientData;
  fOnCloseFunc = onCloseFunc;
  fOnCloseClientData = onCloseClientData;
  fIsCurrentlyAwaitingData = True;
  //这个函数把数据处理函数注册到scheduler中
  doGetNextFrame();
}

//RTP协议相关处理都在这个类里
void MultiFramedRTPSource::doGetNextFrame() {
  if (!fAreDoingNetworkReads) {
    // Turn on background read handling of incoming packets:
    fAreDoingNetworkReads = True;
    //   networkReadHandler -> networkReadHandler1这个回调，用于读取原始的rtp数据
    TaskScheduler::BackgroundHandlerProc* handler = (TaskScheduler::BackgroundHandlerProc*)&networkReadHandler;
    //这个fRTPInterface在RTPSource构造的时候初始化
    fRTPInterface.startNetworkReading(handler);
  }

  fSavedTo = fTo;
  fSavedMaxSize = fMaxSize;
  fFrameSize = 0; // for now
  fNeedDelivery = True;
  //doGetNextFrame1 是用于输出的
  doGetNextFrame1();
}

//挂载回调
void RTPInterface::startNetworkReading(TaskScheduler::BackgroundHandlerProc* handlerProc) {
  // Normal case: Arrange to read UDP packets, 注册读数据回调
  envir().taskScheduler().turnOnBackgroundReadHandling(fGS->socketNum(), handlerProc, fOwner);

  // Also, receive RTP over TCP, on each of our TCP connections:
  // 对于RTP over RTSP的，注册另一个socket的回调
  fReadHandlerProc = handlerProc;
  for (tcpStreamRecord* streams = fTCPStreams; streams != NULL;
       streams = streams->fNext) {
    // Get a socket descriptor for "streams->fStreamSocketNum":
    SocketDescriptor* socketDescriptor = lookupSocketDescriptor(envir(), streams->fStreamSocketNum);

    // Tell it about our subChannel:
    socketDescriptor->registerRTPInterface(streams->fStreamChannelId, this);
  }
}

//真正的RTP读数据回调函数
void MultiFramedRTPSource::networkReadHandler1() {
  BufferedPacket* bPacket = fPacketReadInProgress;
  if (bPacket == NULL) {
    // Normal case: Get a free BufferedPacket descriptor to hold the new network packet:
    // 从fReorderingBuffer里获取一个空的packet
    bPacket = fReorderingBuffer->getFreePacket(this);
  }

  // Read the network packet, and perform sanity checks on the RTP header:
  Boolean readSuccess = False;
  do {
    struct sockaddr_in fromAddress;
    Boolean packetReadWasIncomplete = fPacketReadInProgress != NULL;
    // 调用rtpInterface.handleRead -> recvFrom来读取原始数据到bPacket
    if (!bPacket->fillInData(fRTPInterface, fromAddress, packetReadWasIncomplete)) {
      ...
    }
    if (packetReadWasIncomplete) {
      // We need additional read(s) before we can process the incoming packet:
      fPacketReadInProgress = bPacket;
      return;
    } else {
      fPacketReadInProgress = NULL;
    }

    // Check for the 12-byte RTP header:
    ...

    // Check the Payload Type.
    ...

    // Skip over any CSRC identifiers in the header:
    ...

    // Discard any padding bytes:
    ...

    // The rest of the packet is the usable data.  Record and save it:
    if (rtpSSRC != fLastReceivedSSRC) {
      // The SSRC of incoming packets has changed.  Unfortunately we don't yet handle streams that contain multiple SSRCs,
      // but we can handle a single-SSRC stream where the SSRC changes occasionally:
      fLastReceivedSSRC = rtpSSRC;
      fReorderingBuffer->resetHaveSeenFirstPacket();
    }
    unsigned short rtpSeqNo = (unsigned short)(rtpHdr&0xFFFF);
    Boolean usableInJitterCalculation
      = packetIsUsableInJitterCalculation((bPacket->data()),
						  bPacket->dataSize());
    struct timeval presentationTime; // computed by:
    Boolean hasBeenSyncedUsingRTCP; // computed by:
    //做统计和timestamp更新
    receptionStatsDB()
      .noteIncomingPacket(rtpSSRC, rtpSeqNo, rtpTimestamp,
			  timestampFrequency(),
			  usableInJitterCalculation, presentationTime,
			  hasBeenSyncedUsingRTCP, bPacket->dataSize());

    // Fill in the rest of the packet descriptor, and store it:
    struct timeval timeNow;
    gettimeofday(&timeNow, NULL);
    bPacket->assignMiscParams(rtpSeqNo, rtpTimestamp, presentationTime,
			      hasBeenSyncedUsingRTCP, rtpMarkerBit,
			      timeNow);
    // 根据seq number排序保存packet
    if (!fReorderingBuffer->storePacket(bPacket)) break;

    readSuccess = True;
  } while (0);
  if (!readSuccess) fReorderingBuffer->freePacket(bPacket);

  //因为一般情况下fNeedDelivery在这里不会=true，所以只是进一下下面那个函数
  doGetNextFrame1();
  // If we didn't get proper data this time, we'll get another chance
}

void MultiFramedRTPSource::doGetNextFrame1() {
  while (fNeedDelivery) {
    // If we already have packet data available, then deliver it now.
    Boolean packetLossPrecededThis;
    BufferedPacket* nextPacket
      = fReorderingBuffer->getNextCompletedPacket(packetLossPrecededThis);
    if (nextPacket == NULL) break;

    //走到这里，说明已经收了完整的一个packet
    fNeedDelivery = False;

    if (nextPacket->useCount() == 0) {
      // Before using the packet, check whether it has a special header
      // that needs to be processed:
      ...
    }

    // Check whether we're part of a multi-packet frame, and whether
    // there was packet loss that would render this packet unusable:
    if (fCurrentPacketBeginsFrame) {
      ...
    }

    // The packet is usable. Deliver all or part of it to our caller:
    unsigned frameSize;
    // 通过memcpy，把收到的packet放到fTo里
    nextPacket->use(fTo, fMaxSize, frameSize, fNumTruncatedBytes,
		    fCurPacketRTPSeqNum, fCurPacketRTPTimestamp,
		    fPresentationTime, fCurPacketHasBeenSynchronizedUsingRTCP,
		    fCurPacketMarkerBit);
    fFrameSize += frameSize;

    if (!nextPacket->hasUsableData()) {
      // We're completely done with this packet now
      fReorderingBuffer->releaseUsedPacket(nextPacket);
    }

    if (fCurrentPacketCompletesFrame && fFrameSize > 0) {
      // We have all the data that the client wants.
      if (fNumTruncatedBytes > 0) {
	envir() << "MultiFramedRTPSource::doGetNextFrame1(): The total received frame size exceeds the client's buffer size ("
		<< fSavedMaxSize << ").  "
		<< fNumTruncatedBytes << " bytes of trailing data will be dropped!\n";
      }
      // Call our own 'after getting' function, so that the downstream object can consume the data:
      if (fReorderingBuffer->isEmpty()) {
	// Common case optimization: There are no more queued incoming packets, so this code will not get
	// executed again without having first returned to the event loop.  Call our 'after getting' function
	// directly, because there's no risk of a long chain of recursion (and thus stack overflow):
	afterGetting(this);
      } else {
	// Special case: Call our 'after getting' function via the event loop.
	nextTask() = envir().taskScheduler().scheduleDelayedTask(0,
								 (TaskFunc*)FramedSource::afterGetting, this);
      }
    } else {
      // This packet contained fragmented data, and does not complete
      // the data that the client wants.  Keep getting data:
      fTo += frameSize; fMaxSize -= frameSize;
      fNeedDelivery = True;
    }
  }
}
```

#### 2.4 创建 RTCPInstance

```C++
RTCPInstance::RTCPInstance(UsageEnvironment& env, Groupsock* RTCPgs,
			   unsigned totSessionBW,
			   unsigned char const* cname,
			   RTPSink* sink, RTPSource* source,
			   Boolean isSSMSource)
  : Medium(env), fRTCPInterface(this, RTCPgs), fTotSessionBW(totSessionBW),
    fSink(sink), fSource(source), fIsSSMSource(isSSMSource),
    fCNAME(RTCP_SDES_CNAME, cname), fOutgoingReportCount(1),
    fAveRTCPSize(0), fIsInitial(1), fPrevNumMembers(0),
    fLastSentSize(0), fLastReceivedSize(0), fLastReceivedSSRC(0),
    fTypeOfEvent(EVENT_UNKNOWN), fTypeOfPacket(PACKET_UNKNOWN_TYPE),
    fHaveJustSentPacket(False), fLastPacketSentSize(0),
    fByeHandlerTask(NULL), fByeHandlerClientData(NULL),
    fSRHandlerTask(NULL), fSRHandlerClientData(NULL),
    fRRHandlerTask(NULL), fRRHandlerClientData(NULL),
    fSpecificRRHandlerTable(NULL),
    fAppHandlerTask(NULL), fAppHandlerClientData(NULL) {
  if (fTotSessionBW == 0) { // not allowed!
    env << "RTCPInstance::RTCPInstance error: totSessionBW parameter should not be zero!\n";
    fTotSessionBW = 1;
  }

  if (isSSMSource) RTCPgs->multicastSendOnly(); // don't receive multicast

  double timeNow = dTimeNow();
  fPrevReportTime = fNextReportTime = timeNow;

  fKnownMembers = new RTCPMemberDatabase(*this);
  fInBuf = new unsigned char[maxRTCPPacketSize];
  if (fKnownMembers == NULL || fInBuf == NULL) return;
  fNumBytesAlreadyRead = 0;

  fOutBuf = new OutPacketBuffer(preferredRTCPPacketSize, maxRTCPPacketSize, maxRTCPPacketSize);
  if (fOutBuf == NULL) return;

  if (fSource != NULL && fSource->RTPgs() == RTCPgs) {
    // We're receiving RTCP reports that are multiplexed with RTP, so ask the RTP source
    // to give them to us:
    // 在这里对SimpleRTPSouce注册了fRTCPInstanceForMultiplexedRTCPPackets = rtcpInstance;
    fSource->registerForMultiplexedRTCPPackets(this);
  } else {
    // Arrange to handle incoming reports from the network:
    TaskScheduler::BackgroundHandlerProc* handler
      = (TaskScheduler::BackgroundHandlerProc*)&incomingReportHandler;
    fRTCPInterface.startNetworkReading(handler);
  }

  // Send our first report.
  fTypeOfEvent = EVENT_REPORT;
  onExpire(this);
}
```

###

{% endraw %}
