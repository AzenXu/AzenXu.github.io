---
layout: post
title: 正儿八经学iOS系列 - AVPlayer实现流音频边播边存
---
>边播边下有三套左右实现思路，本文使用AVPlayer + AVURLAsset实现。

#概述
##1. AVPlayer简介
- AVPlayer存在于AVFoundation中，可以播放视频和音频，可以理解为一个**随身听**
- AVPlayer的关联类：
  - AVAsset：一个抽象类，不能直接使用，代表一个要播放的资源。可以理解为一个**磁带**子类AVURLAsset是根据URL生成的包含媒体信息的资源对象。**我们就是要通过这个类的代理实现音频的边播边下的**
  - AVPlayerItem：可以理解为一个**装在磁带盒子里的磁带**

##2. AVPlayer播放原理
- 给播放器设置好想要它播放的URL
- 播放器向URL所在的服务器发送请求，请求两个东西
  - 所需音频片段的起始offset
  - 所需的音频长度
- 服务器根据请求的内容，返回数据
- 播放器拿到数据拼装成文件
- 播放器从拼装好的文件中，找出现在需要播放的片段，进行播放

##3. 边播边下的原理
实现边下边播，其实就是手动实现AVPlayer的上列播放过程。
  - 当播放器需要预先缓存一些数据的时候，不让播放器直接向服务器发起请求，而是向我们自己写的某个类（暂且称之为**播放器的秘书**）发起**缓存请求**
  - **秘书**根据播放器的**缓存请求**的请求内容，向服务器发起请求。
  - 服务器返回**秘书**所需的数据
  - **秘书**把服务器返回的数据写进本地的**缓存文件**中
  - 当需要播放某段声音的时候，向**秘书**发出**播放请求**索要这段音频文件
  - **秘书**从本地的缓存文件中找到播放器**播放请求**所需片段，返回给播放器
  - 播放器拿到数据开心滴播放
  - 当整首歌都缓存完成以后，秘书需要把缓存文件拷贝一份，改个名字，这个文件就是我们所需要的本地持久化文件
  - 下次播放器再播放歌曲的时候，先判断下本地有木有这个名字的文件，有则播放本地文件，木有则向秘书要数据

#技术实现
>OK，边播边下的原理知道了，我们可以正式写代码了~建议先从文末链接处把Demo下载下来，对着Demo咱们慢慢道来~

##1. 类
共需要三个类：
- **MusicPlayerManager**：**CEO**。单例，负责整个工程所有的播放、暂停、下一曲、结束、判断应该播放本地文件还是从服务器拉数据之类的事情
- **RequestLoader**：就是上文所说的**秘书**，负责给播放器提供播放所需的音频片段，以及找人向服务器索要数据
- **RequestTask**：**秘书的小弟**。负责和服务器连接、向服务器请求数据、把请求回来的数据写到本地缓存文件、把写完的缓存文件移到持久化目录去。所有脏活累活都是他做。

##2. 方法
>先从小弟说起

###2.1.  RequestTask
####2.1.0. 概说
如上文所说，小弟是负责做脏活累活的。 负责和服务器连接、向服务器请求数据、把请求回来的数据写到本地缓存文件、把写完的缓存文件移到持久化目录去
####2.1.1. 初始化音频文件持久化文件夹 & 缓存文件
```swift
private func _initialTmpFile() {
        do { try NSFileManager.defaultManager().createDirectoryAtPath(StreamAudioConfig.audioDicPath, withIntermediateDirectories: true, attributes: nil) } catch { print("creat dic false -- error:\(error)") }
        if NSFileManager.defaultManager().fileExistsAtPath(StreamAudioConfig.tempPath) {
            try! NSFileManager.defaultManager().removeItemAtPath(StreamAudioConfig.tempPath)
        }
        NSFileManager.defaultManager().createFileAtPath(StreamAudioConfig.tempPath, contents: nil, attributes: nil)
    }
```

####2.1.2. 与服务器建立连接请求数据
```swift
/**
     连接服务器，请求数据（或拼range请求部分数据）（此方法中会将协议头修改为http）
     
     - parameter offset: 请求位置
     */
    public func set(URL url: NSURL, offset: Int) {
        
        func initialTmpFile() {
            try! NSFileManager.defaultManager().removeItemAtPath(StreamAudioConfig.tempPath)
            NSFileManager.defaultManager().createFileAtPath(StreamAudioConfig.tempPath, contents: nil, attributes: nil)
        }
        _updateFilePath(url)
        self.url = url
        self.offset = offset
        
        //  如果建立第二次请求，则需初始化缓冲文件
        if taskArr.count >= 1 {
            initialTmpFile()
        }
        
        //  初始化已下载文件长度
        downLoadingOffset = 0
        
        //  把stream://xxx的头换成http://的头
        let actualURLComponents = NSURLComponents(URL: url, resolvingAgainstBaseURL: false)
        actualURLComponents?.scheme = "http"
        guard let URL = actualURLComponents?.URL else {return}
        let request = NSMutableURLRequest(URL: URL, cachePolicy: NSURLRequestCachePolicy.ReloadIgnoringCacheData, timeoutInterval: 20.0)
        
        //  若非从头下载，且视频长度已知且大于零，则下载offset到videoLength的范围（拼request参数）
        if offset > 0 && videoLength > 0 {
            request.addValue("bytes=\(offset)-\(videoLength - 1)", forHTTPHeaderField: "Range")
        }
        
        connection?.cancel()
        connection = NSURLConnection(request: request, delegate: self, startImmediately: false)
        connection?.setDelegateQueue(NSOperationQueue.mainQueue())
        connection?.start()
    }
```

####2.1.3. 响应服务器的Response头
```swift
      public func connection(connection: NSURLConnection, didReceiveResponse response: NSURLResponse) {
        isFinishLoad = false
        guard response is NSHTTPURLResponse else {return}
        //  解析头部数据
        let httpResponse = response as! NSHTTPURLResponse
        let dic = httpResponse.allHeaderFields
        let content = dic["Content-Range"] as? String
        let array = content?.componentsSeparatedByString("/")
        let length = array?.last
        //  拿到真实长度
        var videoLength = 0
        if Int(length ?? "0") == 0 {
            videoLength = Int(httpResponse.expectedContentLength)
        } else {
            videoLength = Int(length!)!
        }
        
        self.videoLength = videoLength
        //TODO: 此处需要修改为真实数据格式 - 从字典中取
        self.mimeType = "video/mp4"
        //  回调
        recieveVideoInfoHandler?(task: self, videoLength: videoLength, mimeType: mimeType!)
        //  连接加入到任务数组中
        taskArr.append(connection)
        //  初始化文件传输句柄
        fileHandle = NSFileHandle.init(forWritingAtPath: StreamAudioConfig.tempPath)
    }
```

####2.1.4. 处理服务器返回的数据 - 写入缓存文件中
```swift
    public func connection(connection: NSURLConnection, didReceiveData data: NSData) {
        
        //  寻址到文件末尾
        self.fileHandle?.seekToEndOfFile()
        self.fileHandle?.writeData(data)
        self.downLoadingOffset += data.length
        self.receiveVideoDataHandler?(task: self)
        
//        print("线程 - \(NSThread.currentThread())")
        
        //  注意，这里用子线程有问题
        let queue = dispatch_queue_create("com.azen.taskConnect", DISPATCH_QUEUE_SERIAL)
        dispatch_async(queue) {
//            //  寻址到文件末尾
//            self.fileHandle?.seekToEndOfFile()
//            self.fileHandle?.writeData(data)
//            self.downLoadingOffset += data.length
//            self.receiveVideoDataHandler?(task: self)
//            let thread = NSThread.currentThread()
//            print("线程 - \(thread)")
        }
```

####2.1.5. 服务器文件返回完毕，把缓存文件放入持久化文件夹
```swift
    public func connectionDidFinishLoading(connection: NSURLConnection) {
        func tmpPersistence() {
            isFinishLoad = true
            let fileName = url?.lastPathComponent
//            let movePath = audioDicPath.stringByAppendingPathComponent(fileName ?? "undefine.mp4")
            let movePath = StreamAudioConfig.audioDicPath + "/\(fileName ?? "undefine.mp4")"
            _ = try? NSFileManager.defaultManager().removeItemAtPath(movePath)
            
            var isSuccessful = true
            do { try NSFileManager.defaultManager().copyItemAtPath(StreamAudioConfig.tempPath, toPath: movePath) } catch {
                isSuccessful = false
                print("tmp文件持久化失败")
            }
            if isSuccessful {
                print("持久化文件成功！路径 - \(movePath)")
            }
        }
        
        if taskArr.count < 2 {
            tmpPersistence()
        }
        
        receiveVideoFinishHanlder?(task: self)
    }
```
#### 其他
其他方法包括断线重连以及公开一个cancel方法cancel掉和服务器的连接

###2.2.  RequestTask
####2.2.0. 概说
秘书要干的最主要的事情就是响应播放器老大的号令，所有方法都是围绕着播放器老大来的。秘书需要遵循AVAssetResourceLoaderDelegate协议才能被录用。

####2.2.1. 代理方法，播放器需要缓存数据的时候，会调这个方法
这个方法其实是播放器在说：小秘呀，我想要这段音频文件。你能现在给我还是等等给我啊？
一定要返回：true，告诉播放器，我等等给你。
然后，立马找本地缓存文件里有木有这段数据，有把数据拿给播放器，如果木有，则派秘书的小弟向服务器要。
具体实现代码有点多，这里就不全部贴出来了。可以去看看**文末的Demo**记得赏颗星哟~
```swift
    /**
     播放器问：是否应该等这requestResource加载完再说？
     这里会出现很多个loadingRequest请求， 需要为每一次请求作出处理
     
     - parameter resourceLoader: 资源管理器
     - parameter loadingRequest: 每一小块数据的请求
     
     - returns: <#return value description#>
     */
    public func resourceLoader(resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {
        //  添加请求到队列
        pendingRequset.append(loadingRequest)
        //  处理请求
        _dealWithLoadingRequest(loadingRequest)
        print("----\(loadingRequest)")
        return true
    }
```
####2.2.2. 代理方法，播放器关闭了下载请求
```swift
    /**
     播放器关闭了下载请求
     播放器关闭一个旧请求，都会发起一到多个新请求，除非已经播放完毕了
     
     - parameter resourceLoader: 资源管理器
     - parameter loadingRequest: 待关请求
     */
    public func resourceLoader(resourceLoader: AVAssetResourceLoader, didCancelLoadingRequest loadingRequest: AVAssetResourceLoadingRequest) {
        guard let index = pendingRequset.indexOf(loadingRequest) else {return}
        pendingRequset.removeAtIndex(index)
    }
```

###2.3.  MusicPlayerManager
####2.3.0. 概说
负责调度所有播放器的，负责App中的一切涉及音频播放的事件
唔。。犯个小懒。。代码直接贴上来咯~要赶不上楼下的538路公交啦~~谢谢大家体谅哦~

```swift
public class MusicPlayerManager: NSObject {
    
    
    //  public var status
    
    public var currentURL: NSURL? {
        get {
            guard let currentIndex = currentIndex, musicURLList = musicURLList where currentIndex < musicURLList.count else {return nil}
            return musicURLList[currentIndex]
        }
    }
    
    /**播放状态，用于需要获取播放器状态的地方KVO*/
    public var status: ManagerStatus = .Non
    /**播放进度*/
    public var progress: CGFloat {
        get {
            if playDuration > 0 {
                let progress = playTime / playDuration
                return progress
            } else {
                return 0
            }
        }
    }
    /**已播放时长*/
    public var playTime: CGFloat = 0
    /**总时长*/
    public var playDuration: CGFloat = CGFloat.max
    /**缓冲时长*/
    public var tmpTime: CGFloat = 0
    
    public var playEndConsul: (()->())?
    /**强引用控制器，防止被销毁*/
    public var currentController: UIViewController?
    
    //  private status
    private var currentIndex: Int?
    private var currentItem: AVPlayerItem? {
        get {
            if let currentURL = currentURL {
                let item = getPlayerItem(withURL: currentURL)
                return item
            } else {
                return nil
            }
        }
    }
    
    private var musicURLList: [NSURL]?
    
    //  basic element
    public var player: AVPlayer?
    
    private var playerStatusObserver: NSObject?
    private var resourceLoader: RequestLoader = RequestLoader()
    private var currentAsset: AVURLAsset?
    private var progressCallBack: ((tmpProgress: Float?, playProgress: Float?)->())?
    
    public class var sharedInstance: MusicPlayerManager {
        struct Singleton {
            static let instance = MusicPlayerManager()
        }
        //  后台播放
        let session = AVAudioSession.sharedInstance()
        do { try session.setActive(true) } catch { print(error) }
        do { try session.setCategory(AVAudioSessionCategoryPlayback) } catch { print(error) }
        return Singleton.instance
    }
    
    public enum ManagerStatus {
        case Non, LoadSongInfo, ReadyToPlay, Play, Pause, Stop
    }
}

// MARK: - basic public funcs
extension MusicPlayerManager {
    /**
     开始播放
     */
    public func play(musicURL: NSURL?) {
        guard let musicURL = musicURL else {return}
        if let index = getIndexOfMusic(music: musicURL) {   //   歌曲在队列中，则按顺序播放
            currentIndex = index
        } else {
            putMusicToArray(music: musicURL)
            currentIndex = 0
        }
        playMusicWithCurrentIndex()
    }
    
    public func play(musicURL: NSURL?, callBack: ((tmpProgress: Float?, playProgress: Float?)->())?) {
        play(musicURL)
        progressCallBack = callBack
    }
    
    public func next() {
        currentIndex = getNextIndex()
        playMusicWithCurrentIndex()
    }
    
    public func previous() {
        currentIndex = getPreviousIndex()
        playMusicWithCurrentIndex()
    }
    /**
     继续
     */
    public func goOn() {
        player?.rate = 1
    }
    /**
     暂停 - 可继续
     */
    public func pause() {
        player?.rate = 0
    }
    /**
     停止 - 无法继续
     */
    public func stop() {
        endPlay()
    }
}

// MARK: - private funcs
extension MusicPlayerManager {
    
    private func putMusicToArray(music URL: NSURL) {
        if musicURLList == nil {
            musicURLList = [URL]
        } else {
            musicURLList!.insert(URL, atIndex: 0)
        }
    }
    
    private func getIndexOfMusic(music URL: NSURL) -> Int? {
        let index = musicURLList?.indexOf(URL)
        return index
    }
    
    private func getNextIndex() -> Int? {
        if let musicURLList = musicURLList where musicURLList.count > 0 {
            if let currentIndex = currentIndex where currentIndex + 1 < musicURLList.count {
                return currentIndex + 1
            } else {
                return 0
            }
        } else {
            return nil
        }
    }
    
    private func getPreviousIndex() -> Int? {
        if let currentIndex = currentIndex {
            if currentIndex - 1 >= 0 {
                return currentIndex - 1
            } else {
                return musicURLList?.count ?? 1 - 1
            }
        } else {
            return nil
        }
    }
    
    /**
     从头播放音乐列表
     */
    private func replayMusicList() {
        guard let musicURLList = musicURLList where musicURLList.count > 0 else {return}
        currentIndex = 0
        playMusicWithCurrentIndex()
    }
    /**
     播放当前音乐
     */
    private func playMusicWithCurrentIndex() {
        guard let currentURL = currentURL else {return}
        //  结束上一首
        endPlay()
        player = AVPlayer(playerItem: getPlayerItem(withURL: currentURL))
        observePlayingItem()
    }
    /**
     本地不存在，返回nil，否则返回本地URL
     */
    private func getLocationFilePath(url: NSURL) -> NSURL? {
        let fileName = url.lastPathComponent
        let path = StreamAudioConfig.audioDicPath + "/\(fileName ?? "tmp.mp4")"
        if NSFileManager.defaultManager().fileExistsAtPath(path) {
            let url = NSURL.init(fileURLWithPath: path)
            return url
        } else {
            return nil
        }
    }
    
    private func getPlayerItem(withURL musicURL: NSURL) -> AVPlayerItem {
        
        if let locationFile = getLocationFilePath(musicURL) {
            let item = AVPlayerItem(URL: locationFile)
            return item
        } else {
            let playURL = resourceLoader.getURL(url: musicURL)!  //  转换协议头
            let asset = AVURLAsset(URL: playURL)
            currentAsset = asset
            asset.resourceLoader.setDelegate(resourceLoader, queue: dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0))
            let item = AVPlayerItem(asset: asset)
            return item
        }
    }
    
    private func setupPlayer(withURL musicURL: NSURL) {
        let songItem = getPlayerItem(withURL: musicURL)
        player = AVPlayer(playerItem: songItem)
    }
    
    private func playerPlay() {
        player?.play()
    }
    
    private func endPlay() {
        status = ManagerStatus.Stop
        player?.rate = 0
        removeObserForPlayingItem()
        player?.replaceCurrentItemWithPlayerItem(nil)
        resourceLoader.cancel()
        currentAsset?.resourceLoader.setDelegate(nil, queue: nil)
        
        progressCallBack = nil
        resourceLoader = RequestLoader()
        playDuration = 0
        playTime = 0
        playEndConsul?()
        player = nil
    }
}

extension MusicPlayerManager {
    public override func observeValueForKeyPath(keyPath: String?, ofObject object: AnyObject?, change: [String : AnyObject]?, context: UnsafeMutablePointer<Void>) {
        guard object is AVPlayerItem else {return}
        let item = object as! AVPlayerItem
        if keyPath == "status" {
            if item.status == AVPlayerItemStatus.ReadyToPlay {
                status = .ReadyToPlay
                print("ReadyToPlay")
                let duration = item.duration
                playerPlay()
                print(duration)
            } else if item.status == AVPlayerItemStatus.Failed {
                status = .Stop
                print("Failed")
                stop()
            }
        } else if keyPath == "loadedTimeRanges" {
            let array = item.loadedTimeRanges
            guard let timeRange = array.first?.CMTimeRangeValue else {return}  //  缓冲时间范围
            let totalBuffer = CMTimeGetSeconds(timeRange.start) + CMTimeGetSeconds(timeRange.duration)    //  当前缓冲长度
            tmpTime = CGFloat(tmpTime)
            print("共缓冲 - \(totalBuffer)")
            let tmpProgress = tmpTime / playDuration
            progressCallBack?(tmpProgress: Float(tmpProgress), playProgress: nil)
        }
    }
    
    private func observePlayingItem() {
        guard let currentItem = self.player?.currentItem else {return}
        //  KVO监听正在播放的对象状态变化
        currentItem.addObserver(self, forKeyPath: "status", options: NSKeyValueObservingOptions.New, context: nil)
        //  监听player播放情况
        playerStatusObserver = player?.addPeriodicTimeObserverForInterval(CMTimeMake(1, 1), queue: dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), usingBlock: { [weak self] (time) in
            guard let `self` = self else {return}
            //  获取当前播放时间
            self.status = .Play
            let currentTime = CMTimeGetSeconds(time)
            let totalTime = CMTimeGetSeconds(currentItem.duration)
            self.playDuration = CGFloat(totalTime)
            self.playTime = CGFloat(currentTime)
            print("current time ---- \(currentTime) ---- tutalTime ---- \(totalTime)")
            self.progressCallBack?(tmpProgress: nil, playProgress: Float(self.progress))
            if totalTime - currentTime < 0.1 {
                self.endPlay()
            }
            }) as? NSObject
        //  监听缓存情况
        currentItem.addObserver(self, forKeyPath: "loadedTimeRanges", options: NSKeyValueObservingOptions.New, context: nil)
    }
    
    private func removeObserForPlayingItem() {
        guard let currentItem = self.player?.currentItem else {return}
        currentItem.removeObserver(self, forKeyPath: "status")
        if playerStatusObserver != nil {
            player?.removeTimeObserver(playerStatusObserver!)
            playerStatusObserver = nil
        }
        currentItem.removeObserver(self, forKeyPath: "loadedTimeRanges")
    }
}

public struct StreamAudioConfig {
    static let audioDicPath: String = NSSearchPathForDirectoriesInDomains(NSSearchPathDirectory.DocumentDirectory, NSSearchPathDomainMask.UserDomainMask, true).last! + "/streamAudio"  //  缓冲文件夹
    static let tempPath: String = audioDicPath + "/temp.mp4"    //  缓冲文件路径 - 非持久化文件路径 - 当前逻辑下，有且只有一个缓冲文件
    
}
```

[iOS音频边播边下Demo，戳这里~](https://github.com/AzenXu/AZXMusicPlayerManager)

如果对大家有点小帮助，记得赏颗Star哦~ 谢啦~😋