# FFplay.c

![5cf2302a3446f48597](https://i.loli.net/2019/06/01/5cf2302a3446f48597.png)



## 数据处理



**main线程**

    用于处理事件和刷新图像帧到SDL进行展示



**read_thread线程**

    用于读取原始包到数据队列中



**video_thread线程**

    用于从数据队列中读取包，然后解码成帧，以便播放视频



**audio_thread线程**

    用于从数据队列中读取包，然后解码成帧，以便播放声音



## main()

主函数

`avformat_network_init()`

初始化网络库()This is optional, and not recommended anymore.

[http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#ga84542023693d61e8564c5d457979c932](http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#ga84542023693d61e8564c5d457979c932)

`show_banner()`

打印输出FFmpeg版本信息（编译时间，编译选项，类库信息等）

`parse_options()` 解析命令行输入选项

1. `parse_option()`解析参数

2. `find_option()_`根据参数找到对应的`OptionDef`

3. `write_option()_`执行`OptionDef`

`SDL_Init()` 初始化SDL，FFPlay中的视频与音频都是使用到了SDL

`SDL_CreateWindow()`创建SDL窗口

[https://wiki.libsdl.org/SDL_CreateWindow](https://wiki.libsdl.org/SDL_CreateWindow)

`SDL_CreateRedner()`为SDL窗口创建SDL渲染上下文

[https://wiki.libsdl.org/SDL_CreateRenderer](https://wiki.libsdl.org/SDL_CreateRenderer)

[https://wiki.libsdl.org/SDL_RendererFlags](https://wiki.libsdl.org/SDL_RendererFlags)

## stream_open()

打开输入媒体

`frame_queue_init()`初始化帧队列，`FrameQueue`内部是个有限数组，内部根据`max_size_`来初始化默认缓存数据

`f->queue[i].frame = av_frame_alloc()`

`frame_queue_destory()`销毁帧队列，释放初始化时的缓存数据`av_frame_free()`

`packet_queue_init()`初始化包队列，`PacketQueue`内部是个自定义链表

`init_clock()`初始化时钟，使用默认值(`NAN/-1`)来初始化时钟的`pst/last_update/pst_dirft/serial_`等数据。

1. `av_gettime_relative()`从未指明的某个点获取到现在的时间（纳秒）[https://ffmpeg.org/doxygen/3.2/time_8c.html#adf0e36df54426fa167e3cc5a3406f3b7](https://ffmpeg.org/doxygen/3.2/time_8c.html#adf0e36df54426fa167e3cc5a3406f3b7)

2. `set_clock_at(c,pts,serial,time)`

## read_thread()

帧读取线程

`avformat_alloc_context()`初始化上下文

[http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#gac7a91abf2f59648d995894711f070f62](http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#gac7a91abf2f59648d995894711f070f62)

`avformat_open_input()`打开文件流，读取文件头部数据

`avformat_find_stream_info()`获取媒体流信息

`avformat_seek_file()`

`av_dump_format()`输出媒体信息到控制台

`av_find_best_stream()`获取最合适的流

`av_guess_sample_aspect_ratio()`获取流采样率

`AVCodecParameters`编解码器参数

`set_default_window_size()`设置窗口大小

1. `calculate_display_rect()`根据屏幕宽高和流像素宽高计算出可显示区域

`stream_component_open()`分别打开视频/音频/字幕解码线程

`av_read_pause()/av_read_play()`暂停或者播放网络流

[http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga27db687592d99f25ccf81a3b3ee8da9c](http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga27db687592d99f25ccf81a3b3ee8da9c)

`avformat_seek_file()`快进

[http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga3b40fc8d2fda6992ae6ea2567d71ba30](http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga3b40fc8d2fda6992ae6ea2567d71ba30)

`packet_queue_flush()`清空包队列缓存

`stream_has_enough_packets()`是否有足够的包缓存

`av_read_frame()`读取一帧流原始数据

`packet_queue_put()`将读取的包数据加入到队列中

## stream_component_open()

分别打开视频/音频/字幕解码线程

`avcodec_alloc_context3()` 创建编解码器上下文

`avcodec_parameters_to_context()`从流中复制参数到编解码器上下文中

`avcodec_find_decoder()`根据编码ID获取编解码器

`avcodec_find_decoder_by_name()`根据名称查找编解码器

`avcodec_open2()` 根据编解码器初始化编解码器上下文

`audio_open()`打开音频解码

1. `av_get_default_channel_layout(wanted_nb_channels)`

2. `av_get_channel_layout_nb_channels(wanted_channel_layout)`

3. `sdl_audio_callback()`
   
   1. `audio_decode_frame()`
      
      1. `synchronize_audio()`
      
      2. `swr_convert()`

4. `SDL_OpenAudioDevice()`[https://wiki.libsdl.org/SDL_OpenAudioDevice](https://wiki.libsdl.org/SDL_OpenAudioDevice)

`audio_thread()`解码音频帧

1. `decoder_decode_frame()`
   
   1. `avcodec_send_packet()`
   
   2. `avcodec_receive_frame()`

`video_thread()`解码视频帧

1. `get_video_frame()`获取视频帧
   
   1. `decoder_decode_frame()`
   
   2. `av_guess_sample_aspect_ratio()`

2. `queue_picture()` 入队视频图像数据
   
   1. `frame_queue_peek_writable()` 从帧缓存区中等待获取一个可用数据
   
   2. `av_frame_move_ref()`转移引用
   
   3. `frame_queue_push()`增加帧队列size

`subtitle_thread()_`_解码字幕帧

1. `frame_queue_peek_writable()`

2. `decoder_decode_frame()`

3. `frame_queue_push()`

## event_loop()

处理键盘事件与视频刷新

`video_refresh()`处理视频刷新与显示

1. `frame_queue_peek_last()`出队图像

2. `frame_queue_peek()`

3. `video_display()`显示图像
   
   1. `video_open()`开打窗口
   
   2. `video_image_display()`图像展示
      
      1. `frame_queue_peek_last()`
      
      2. `calculate_display_rect()`计算显示画面的位置。当拉伸了SDL的窗口的时候，可以让其中的视频保持纵横比
      
      3. `upload_texture()`
      
      4. `SDL_RenderCopyEx()`
