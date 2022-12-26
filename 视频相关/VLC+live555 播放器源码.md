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
    /* Create a task that will be called if we wait more than 300ms */
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

#### 1.2 调度器启动





### 1 MediaSubsession::initiate

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



### 2 SimpleRTPSource::createNew





{% endraw %}