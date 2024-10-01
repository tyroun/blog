{% raw %}

# ZLM 代码阅读

##epoll

### epoll 和 Select/poll 区别

|     \      |                        select                         |                        poll                         |                                                 epoll                                                  |
| :--------: | :---------------------------------------------------: | :-------------------------------------------------: | :----------------------------------------------------------------------------------------------------: |
|  操作方式  |                         遍历                          |                        遍历                         |                                                  回调                                                  |
|  底层实现  |                         数组                          |                        链表                         |                                                 哈希表                                                 |
|  IO 效率   |       每次调用都进行线性遍历，时间复杂度为 O(n)       |      每次调用都进行线性遍历，时间复杂度为 O(n)      | 事件通知方式，每当 fd 就绪，系统注册的回调函数就会被调用，将就绪 fd 放到 rdllist 里面。时间复杂度 O(1) |
| 最大连接数 |               1024（x86）或 2048（x64）               |                       无上限                        |                                                 无上限                                                 |
|  fd 拷贝   | 每次调用 select，都需要把 fd 集合从用户态拷贝到内核态 | 每次调用 poll，都需要把 fd 集合从用户态拷贝到内核态 |                     调用 epoll_ctl 时拷贝进内核并保存，之后每次 epoll_wait 不拷贝                      |

**传统 select/poll 的另一个致命弱点就是当你拥有一个很大的 socket 集合，由于网络得延时，使得任一时间只有部分的 socket 是"活跃" 的，而 select/poll
每次调用都会线性扫描全部的集合，导致效率呈现线性下降。**

**但是 epoll 不存在这个问题，它只会对"活跃"的 socket 进 行操作**---这是因为在内核实现中 epoll 是根据每个 fd 上面的 callback 函数实现的。于是，只有"活跃"的 socket 才会主动去调用
callback 函数，其他 idle 状态的 socket 则不会，在这点上，epoll 实现了一个<font color="pink"> "伪"AIO</font>，因为这时候推动力在 os 内核。

### epoll 原理

#### 1 将控制与阻塞分离。

epoll 精巧的使用了 3 个方法来实现 select 方法要做的事：

1. 新建 epoll 描述符==epoll_create()
2. epoll_ctrl(epoll 描述符，添加或者删除所有待监控的连接)
3. 返回的活跃连接 ==epoll_wait（ epoll 描述符 ）
4. epoll_ctrl 是不太频繁调用的，而 epoll_wait 是非常频繁调用的。这时，epoll_wait 却几乎没有入参，这比 select
   的效率高出一大截，而且，它也不会随着并发连接的增加使得入参越发多起来，导致内核执行效率下降

#### 2 mmap

内核和用户态操作的 fd，都是同一块内存

#### 3 RBTree

1. epoll_ctl 会在一颗 RBtree 上添加删除 event，以 fd 排序

2. 中断发生，fd 状态改变，fd 的回调函数 ep_poll_callback 被调用

3 /ep_poll_callback 将相应 fd 对应 epitem 加入双向链表 rdlist

4. epoll_wait 发现 rdlist 不为空，就被唤醒继续执行

5. ep_events_transfer 函数将 rdlist 中的 epitem 拷贝到 txlist 中，并将 rdlist 清空。

6. ep_send_events 函数（很关键），它扫描 txlist 中的每个 epitem，调用其关联 fd 对用的 poll 方法。此时对 poll 的调用仅仅是取得 fd 上较新的 events（防止之前 events
   被更新），之后将取得的 events 和相应的 fd 发送到用户空间（封装在 struct epoll_event，从 epoll_wait 返回）

### epoll 使用

```c
int epoll_create ( int size );

/*
函数说明：
     fd：要操作的文件描述符
     op：指定操作类型
操作类型：
     EPOLL_CTL_ADD：往事件表中注册fd上的事件
     EPOLL_CTL_MOD：修改fd上的注册事件
     EPOLL_CTL_DEL：删除fd上的注册事件
*/
int epoll_ctl ( int epfd, int op, int fd, struct epoll_event *event );

struct epoll_event
{
    __unit32_t events;    // epoll事件
    epoll_data_t data;     // 用户数据
};

typedef union epoll_data
{
    void* ptr;              //指定与fd相关的用户数据
    int fd;                 //指定事件所从属的目标文件描述符
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

//events：检测到事件，将所有就绪的事件从内核事件表中复制到它的第二个参数events指向的数组中
int epoll_wait (int epfd,struct epoll_event* events,int maxevents,int timeout );
```

### LT 与 ET 模式

LT：LT(level triggered)是缺省的工作方式，并且同时支持 block 和 no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 IO 操作。**
如果你不作任何操作，内核还是会继续通知你的**。传统的 select/poll 都是这种模型的代表

ET：ET (edge-triggered) 是高速工作方式，只支持 no-block socket(非阻塞)。 在这种模式下，**当描述符从未就绪变为就绪时，内核就通过 epoll
告诉你，然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的 就绪通知**，直到你做了某些操作而导致那个文件描述符不再是就绪状态(比如 你在发送，接收或是接受请求，或者发送接收的数据少于一定量时导致了一个
EWOULDBLOCK 错误)。但是请注意，如果一直不对这个 fd 作 IO 操作(从而导致它再次变成未就绪)，内核就不会发送更多的通知(only once)。不过在 TCP 协议中，ET 模式的加速效用仍需要更多的 benchmark
确认

## ZLM 中的 ZLToolKit

### Poller

EventPoller

```c++
//封装了epoll
Class EventPoller : public TaskExecutor{
	//构造函数
    EventPoller(ThreadPool::Priority priority ){
        //调用epoll_create创建_epoll_fd
        _epoll_fd = epoll_create(EPOLL_SIZE);

        //添加内部管道事件
        //不同线程要触发epoll_wait时，对着_pipe写，然后把回调放入_list_task。在onPipeEvent中取出执行
        addEvent(_pipe.readFD(), Event_Read, [this](int event) { onPipeEvent(); }) == -1)
    }

    //添加事件监听
    int addEvent(int fd, int event, PollEventCB cb){
         if (isCurrentThread()) {
             //添加事件
             epoll_ctl(_epoll_fd, EPOLL_CTL_ADD, fd, &ev);
             //把输入的回调加入_event_map
             _event_map.emplace(fd, std::make_shared<PollEventCB>(std::move(cb)));
         }
         //如果不是run_loop这个线程添加的event，用async()函数把当前这个addEvent添加到内部管道事件，通过写_pipe通知run_loop线程执行
         async(...)
    }


    Task::Ptr EventPoller::async_l(...){
         _list_task.emplace_back(ret); //添加入_list_task
    	  _pipe.write("", 1);//写数据到管道,唤醒主线程
    }

    void runLoop(bool blocked,bool regist_self){
        if (blocked) {
            while (!_exit_flag) {
                int ret = epoll_wait(_epoll_fd, events, EPOLL_SIZE, minDelay ? minDelay : -1);
                for (int i = 0; i < ret; ++i) {
                    auto it = _event_map.find(fd);
                    auto cb = it->second;
                    (*cb)(toPoller(ev.events)); //执行回调
                }

            }
        }else{
            //新开线程
            _loop_thread = new thread(&EventPoller::runLoop, this, true, regist_self);
        }
    }
}
```

EventPollerPool - 这个是有多个线程在 epoll_wait 的线程池

```c++
class EventPollerPool : public TaskExecutorGetterImp {
    EventPollerPool() {
        auto size = addPoller("event poller", s_pool_size, ThreadPool::PRIORITY_HIGHEST, true);
    }

    addPoller(const string &name,size_t size,...){
        for (size_t i = 0; i < size; ++i) {
            //创建EventPoller
            EventPoller::Ptr poller(new EventPoller((ThreadPool::Priority) priority));
            //run loop，开一个epool_wait的loop线程
       		poller->runLoop(false, register_thread);
            //异步设置线程信息
            poller->async([i, cpus, full_name]() {
                setThreadName(full_name.data());
                setThreadAffinity(i % cpus);
            });
            _threads.emplace_back(std::move(poller)); //_threads表示可执行的线程池
        }
    }

}
```

### Thread

TaskExecutor

就是提供了 async 和 sync 方法，EventPoller 通过 epoll 实现了 async 方法，sync 方法就是直接调用回调函数，只是对异常做了以下处理

TaskExecutorGetterImp

就是维护\_threads 里面的所有 TaskExecutor

ThreadPool - 这个是直接执行 task 的线程池

```c++
class ThreadPool : public TaskExecutor {
    ThreadPool(int num = 1, Priority priority = PRIORITY_HIGHEST, bool auto_run = true){
        if (auto_run) {
            start(); //启动线程
        }
    }

    async(TaskIn task, bool may_sync = true){
        _queue.push_task(ret); //_queue维护一个回调函数map
    }

    //start以后，就从_queue里拿出所有task，每个开一个线程执行
    void start() {
        for (size_t i = 0; i < total; ++i) {
            _thread_group.create_thread(bind(&ThreadPool::run, this));
        }
    }

    void run() {
        Task::Ptr task;
        while (true) {
            startSleep();
            if (!_queue.get_task(task)) {
                //空任务，退出线程
                break;
            }
            sleepWakeUp();
            (*task)();
            task = nullptr;

        }
    }

}
```

### Network

Socket - 负责操作实际的 socket 调用流程，最终调用到 TCPServer 注册来的回调

TCPServer - 负责注册 socket 的创建，监听回调函数

## RTP Server 收到流

### 1 WebApi.cpp/openRtpServer

获取 stream_id，默认 stream_id 有值且 enable_tcp=1

创建 RtpServer\* server

server->start

server->setOnDetach 注册 detach 时的回调

在 s_rtpServerMap 里注册这个 server

val["port"] = server->getPort() //回复 json

### 2 RtpServer::start

创建 Socket

创建并启动 TCP Server

启动 rtcp

对 rtp_socket 注册回调 setOnRead

​ 调用 inputRtp 解析 Rtp

​ 调用 onRecvRtp 回复 rtcp

### 3 RtpProcess::inputRtp

如果第一次启动，调用 emitOnPublish

如果 config 了 kDumpDir，那么把 rtp 写到\_save_file_rtp

创建 GB28181Process 的\_process

调用\_process->inputRtp

### 4 RtpProcess::emitOnPublish

建立 MediaSource

创建 MultiMediaSourceMuxer 类，构造函数里面构建了多个 Muxer(\_rtmp/\_rtsp/\_hls/\_mp4/\_ts/\_fmp4)

### 4 GB28181Process::inputRtp

产生\_rtp_receiver=RtpReceiverImp

1. 注册`GB28181Process::onRtpSorted`回调，表示解析出一个完整的 Rtp 包时，调用\_rtp_decoder->inputRtp

根据不同的 RtpHeader->pt，产生不同的\_rtp_decoder。265 的话就是 H265RtpDecoder

\_interface->addTrack 就是调用了 `RtpProcess::addTrack`，这里面把这个流加入到了 MultiMediaSourceMuxer 里面

\_rtp_decoder->addDelegate 注册了 frame 的回调。收到一帧 frame 后，会调用`GB28181Process::onRtpDecode`

调用 RtpReceiverImp::inputRtp -> RtpTrack::inputRtp 解析收到包

### 5 RtpTrack::inputRtp

解析 RTP header，比对 ssrc

RtpPacket::create()创建一个 RtpPacket

出来头 4 个 byte 填了 rtp over tcp 头，其他 memcpy

调用 sortPacket

1. 根据 seq 排序，然后放入\_pkt_sort_cache_map
2. tryPopPacket()尝试是否可以 pop 出一个完整的 packet
    1. popPacket() -> popIterator() 弹出 packet，调用 RtpTrackImp::OnSorted 回调
    2. RtpTrackImp::OnSorted 在 RtpReceiverImp 构造时传入，就是`GB28181Process::onRtpSorted` -> H265RtpDecoder::inputRtp

### 6 H265RtpDecoder::inputRtp

调用 H265RtpDecoder::decodeRtp

根据不同的 nal，调用 H265RtpDecoder::unpackAp

--> H265RtpDecoder::singleFrame

--> H265RtpDecoder::outputFrame

--> RtpCodec::inputFrame

--> 会调用到 FrameDispatcher 里注册的\_delegates_write 的 inputFrame(frame)，这个函数就是上面注册的`GB28181Process::onRtpDecode`

把这个 frame 送走以后，需要重新\_frame = obtainFrame()，初始化一个帧结构用来收帧

### 7 GB28181Process::onRtpDecode

第一次进入创建\_decoder，PS 流就创建 PSDecoder

调用\_decoder->input

PSDecoder::input -> HttpRequestSplitter::input -> PSDecoder::onSearchPacketTail -> ps_demuxer_input

再调用到以下回调

### 8 PSDecoder 设置回调的方法

在构造函数里，调用 libmpeg 的库函数 ps_demuxer_create 创建\_ps_demuxer。在里面注册了 onpacket 的回调函数。这个函数会再回调 PSDecoder->
\_on_decode。这个\_on_decode 是由 PSDecoder::setOnDecode 注册的，是在 DecoderImp 构造时注册的 DecoderImp::onDecode

简单表示

PSDecoder::PSDecoder()

-> _ps_demuxer=ps_demuxer_create_

-> PSDecoder->\_on_decode

-> PSDecoder::setOnDecode

-> DecoderImp::DecoderImp()

-> DecoderImp::onDecode

同样的方法也设置了 ps_demuxer 的 notify=DecoderImp::onStream。Notify 表示同一个 RTP 流内部视频或音频流有变化的情况

### 9 DecoderImp::onDecode

这个是解析出一个 PS packet 的时候会调用的

如果没有这个 tracks，那么调用 DecoderImp::onTrack 在 sink 里添加这个 tracks

FrameMerger::inputFrame 合流

DecoderImp::onFrame --> \_sink->inputFrame 输入

这个\_sink 就是 GB28181Process 中的\_interface，就是 RtpProcess

### 10 RtpProcess::inputFrame

调用*muxer->inputFrame() *

--> MediaSink::inputFrame() 里面从\_track_map 获取 Track

--> 默认是 H264Track::inputFrame()

--> H264Track::inputFrame_l()

如果 SPS/PPS 先保存，遇到 I 帧先插入 SPS/PPS frame,再送后一个 frame

--> VideoTrack/FrameDispatcher::inputFrame

--> FrameWriterInterfaceHelper::inputFrame 去调用回调

\_-->\_MediaSink.cpp:41

​ 这个回调当前面 RtpProcess 里调用了 addTrack 时

​ --> MediaSink::addTrack

​ --> track->addDelegate 这里会注册这个回调。

​ 调用 MultiMediaSourceMuxer::onTrackFrame

​ 在 MediaSink 中维护了一个*frame_unread，保留了一个未读的 Frame。把该帧存入* frame_unread

### 11 MultiMediaSourceMuxer::onTrackFrame

调用 RtpSender::inputFrame

这里就开始进入 send RTP 流程

## RTP send 流程

### 1 WebApi.cpp/startSendRtp

```c++
api_regist("/index/api/startSendRtp",[](API_ARGS_MAP_ASYNC){
    //入口
    auto src = MediaSource::find(allArgs["vhost"], allArgs["app"], allArgs["stream"], allArgs["from_mp4"].as<int>());
    //检查MediaSource
    src->startSendRtp(allArgs["dst_url"], allArgs["dst_port"], allArgs["ssrc"], allArgs["is_udp"], allArgs["src_port"], [val, headerOut, invoker](uint16_t local_port, const SockException &ex) mutable{
        //这段是start成功以后，返回http response
            if (ex) {
                val["code"] = API::OtherFailed;
                val["msg"] = ex.what();
            }
            val["local_port"] = local_port;
            invoker(200, headerOut, val.toStyledString());
        });

}
```

###2 MediaSource.cpp

```c++
// 1 先进这个函数
void MediaSource::startSendRtp(const string &dst_url, uint16_t dst_port, const string &ssrc, bool is_udp, uint16_t src_port, const function<void(uint16_t local_port, const SockException &ex)> &cb){
    auto listener = _listener.lock();
    if (!listener) {
        cb(0, SockException(Err_other, "尚未设置事件监听器"));
        return;
    }
    //listener = MediaSourceEventInterceptor
    return listener->startSendRtp(*this, dst_url, dst_port, ssrc, is_udp, src_port, cb);
}

// 2 再进这个函数
void MediaSourceEventInterceptor::startSendRtp(MediaSource &sender, const string &dst_url, uint16_t dst_port, const string &ssrc, bool is_udp, uint16_t src_port, const function<void(uint16_t local_port, const SockException &ex)> &cb){
    auto listener = _listener.lock();
    if (listener) {
        // 3 从这里进muxer
        listener->startSendRtp(sender, dst_url, dst_port, ssrc, is_udp, src_port, cb);
    } else {
        MediaSourceEvent::startSendRtp(sender, dst_url, dst_port, ssrc, is_udp, src_port, cb);
    }
}

```

### 3 MultiMediaSourceMuxer.cpp

```c++
void MultiMediaSourceMuxer::startSendRtp(MediaSource &, const string &dst_url, uint16_t dst_port, const string &ssrc, bool is_udp, uint16_t src_port, const function<void(uint16_t local_port, const SockException &ex)> &cb){

    RtpSender::Ptr rtp_sender = std::make_shared<RtpSender>(atoi(ssrc.data()));
    weak_ptr<MultiMediaSourceMuxer> weak_self = shared_from_this();

    // 4 从这里进RtpSender
    rtp_sender->startSend(dst_url, dst_port, is_udp, src_port, [weak_self, rtp_sender, cb, ssrc](uint16_t local_port, const SockException &ex) {
        cb(local_port, ex);
        auto strong_self = weak_self.lock();
        if (!strong_self || ex) {
            return;
        }
        for (auto &track : strong_self->getTracks(false)) {
            rtp_sender->addTrack(track);
        }
        rtp_sender->addTrackCompleted();
        lock_guard<mutex> lck(strong_self->_rtp_sender_mtx);
        strong_self->_rtp_sender[ssrc] = rtp_sender; //加入_rtp_sender，等收到frame后就会调用rtp_sender->inputFrame
    });
}
```

### 4 RtpSender.cpp

```c++
void RtpSender::startSend(const string &dst_url, uint16_t dst_port, bool is_udp, uint16_t src_port, const function<void(uint16_t local_port, const SockException &ex)> &cb){

    _is_udp = is_udp;
    _socket = Socket::createSocket(_poller, false);
    _dst_url = dst_url;
    _dst_port = dst_port;
	_src_port = src_port;
    weak_ptr<RtpSender> weak_self = shared_from_this();
    if (is_udp) {
        _socket->bindUdpSock(src_port);
        auto poller = _poller;
        auto local_port = _socket->get_local_port();
        WorkThreadPool::Instance().getPoller()->async([cb, dst_url, dst_port, weak_self, poller, local_port]() {
            struct sockaddr addr;
            //切换线程目的是为了dns解析放在后台线程执行
            if (!SockUtil::getDomainIP(dst_url.data(), dst_port, addr)) {
                poller->async([dst_url, cb, local_port]() {
                    //切回自己的线程
                    cb(local_port, SockException(Err_dns, StrPrinter << "dns解析域名失败:" << dst_url));
                });
                return;
            }

            //dns解析成功
            poller->async([addr, weak_self, cb, local_port]() {
                //切回自己的线程
                cb(local_port, SockException());
                auto strong_self = weak_self.lock();
                if (strong_self) {
                    strong_self->_socket->bindPeerAddr(&addr);
                    strong_self->onConnect();
                }
            });
        });
    } else {
        ...
}
```

### 5 RtpSender::inputFrame

MultiMediaSourceMuxer 接收到一个 Frame 之后，会从 MultiMediaSourceMuxer::onTrackFrame 进入送帧操作

--> MpegMuxer::inputFrame

--> FrameMerger::inputFrame

    	1. WillFlush(frame) 判断是否需要flush，只要有1帧就需要flush

2. 如果\_frame_cache 有多帧，会把多个帧用 BufferLikeString 和合成 1 帧 merged_frame
3. 调用外部输入的回调，MPEG.cpp:67，调用 libmpeg 的库函数`mpeg_muxer_input--> ps_muxer_input()`。这个函数里会把这些 frame 都打上 PS 流的各种头，PES 头是根据最大
   packet 长度包的。
4. ps_muxer_input 把封装成 PS 流的 packet 送入 write 的回调函数。MPEG.cpp:111 。 这个回调在 MpegMuxer::createContext 初始化
5. 调用 MpegMuxer::onWrite_l --> PSEncoderImp::onWrite --> CommonRtpEncoder::inputFrame 开始封装 RTP 流

### 6 CommonRtpEncoder::inputFrame

#### 6.1 封装 RTP 流

每个 RTP 包的最大 size，在 config.ini 里面 videoMtuSize=1400 定义了 mtu 的 size，

```c++
//返回rtp负载最大长度
size_t getMaxSize() const {
    // RtpPacket::kRtpHeaderSize = 12
    return _mtu_size - RtpPacket::kRtpHeaderSize;
}
```

把 packet 拆解成多个 mtu 后，调用 RtpInfo::makeRtp 把 raw data 封装成 Rtp

mark 只在一个 packet 拆成最后 1 个 Rtp 包的时候才打上

#### 6.2 发送

对每个 Rtp 包调用 RtpCodec::inputRtp

--> RingBuffer::write

--> RingDelegateHelper::onWrite

--> 回调再 PSEncoder.cpp:24

--> RtpCachePS::onRTP

--> RtpCache::input

把 packet 放入 PacketCache.\_cache

--> PacketCache::flushAll

--> RtpCache::onFlush

--> 调用回调 RtpSender.cpp:22

--> RtpSender::onFlushRtpList

在这里用 socket->write 送帧

{% endraw %}
