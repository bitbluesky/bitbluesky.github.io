# 背景
因为GPU解码输出的像素格式是NV12，而NV12转换BGR24的耗时比YUV420转换BGR24要高4倍，因此使用scale_npp在GPU上将像素格式转为YUV420再输出。

同时，也需要使用fps filter来设置帧率。

同样使用FFmpeg的api，类似功能是命令行如下：

ffmpeg -hwaccel cuda -hwaccel_output_format cuda -i ~/video/test.mp4 -vf "fps=15,scale_npp=format=yuv420p,hwdownload,format=yuv420p" -f null /dev/null

## 报错现象
出错先打印下面的日志，应该是decoder的某个索引用完了，导致send packet出错，内部又不断的重复初始化，显存也就耗光了。
2021-06-09 12:14:42,473 FATAL 140468490848000 xxxx.cpp ffmpeg_log_callback No decoder surfaces left

运行一段时间后日志的报错： 同时nvidia-smi查看显存占用，发现显存已经被占满。

2021-06-09 12:51:30,353 FATAL 140464455923456 xxxx.cpp ffmpeg_log_callback decoder->cvdl->cuvidCreateDecoder(&decoder->decoder, params) failed
2021-06-09 12:51:30,353 FATAL 140464455923456 xxxx.cpp ffmpeg_log_callback -> CUDA_ERROR_OUT_OF_MEMORY: out of memory
2021-06-09 12:51:30,353 FATAL 140464455923456 xxxx.cpp ffmpeg_log_callback

2021-06-09 12:51:30,353 FATAL 140464455923456 xxxx.cpp ffmpeg_log_callback Failed setup for format cuda: hwaccel initialisation returned error.

2021-06-09 12:51:30,353 NOTICE 140464455923456 xxxx.cpp get_hw_format Failed to get HW surface format.
2021-06-09 12:51:30,353 FATAL 140464455923456 xxxx.cpp ffmpeg_log_callback decode_slice_header error



# 原因
经过测试，fps=12.5得设置在scale_npp后面才行。设置在前面就会有显存问题。可能是解码和npp都在显存上处理，设置framerate的filter插入在npp之前，丢掉的frame没有真正释放显存。

fps, as a filter, needs to be inserted in a filtergraph. It offers five rounding modes that affect which source frames are dropped or duplicated in order to achieve the target framerate.

# 解决方案
修改init_filters时设置给avfilter_graph_parse_ptr的参数，将filters_descr从

fps=12.5,scale_npp=format=yuv420p,hwdownload,format=yuv420p

改为

scale_npp=format=yuv420p,hwdownload,format=yuv420p,fps=12.5

备注：调整filters_descr后，因为fps filter后移，可能会对效率有一定影响。

# 排查步骤
## 复现问题
经过多次测试，发现启动三个进程后，用postman给每个进程批量发送25路rtmp视频流并发，3-5分钟后即可复现。

## 确定导致出错的范围
1. 查看日志报错信息，进行汇总，发现首先出现的异常是No decoder surfaces left，正常情况不应该有这个报错。

2. 添加调试日志

3. 临时替换掉ffmpeg filter的代码，直接调用av_hwframe_transfer_data将解码结果拷贝回内存，测试发现没有出现问题。

4. 改回ffmpeg filter进行像素格式转换，复现问题。

5. 针对ffmpeg filter，修改filters_descr，去除fps的过滤进行测试，结果正常。因此出错和fps filter有关。

6. 尝试替换新的fps过滤方案。同时将filters_descr中的fps=后移，测试结果也正常。结合之前的测试结果，应该是fps filter插入到scale_npp之前时，缩小帧率会drop frame，但是显存没有正确释放。

TODO，尝试fps=在scale_npp之前时修复显存泄漏的问题。得深入看FFmpeg fps filter的代码。

## 其他，

一路并发，解码进程会占用205MB显存。
75路并发时，三个显卡各占用5128MB显存。

## fps的问题

解码时设置framerate的filter，fps=12.5， 处理完的tmp frame的pts就是加1递增了。之前frame的pts是间隔40ms。

![image](https://user-images.githubusercontent.com/6881931/121341292-18c58b00-c953-11eb-8986-02b306991bb7.png)


不设置fps=xxx测试， npp scale像素转换的输出pts也是间隔40ms；

![image](https://user-images.githubusercontent.com/6881931/121341300-1bc07b80-c953-11eb-9b2c-16e88e4af424.png)


# 参考信息
How do I reduce frames with blending in ffmpeg

https://stackoverflow.com/questions/22547253/how-do-i-reduce-frames-with-blending-in-ffmpeg

Changing the frame rate

https://trac.ffmpeg.org/wiki/ChangingFrameRate

Framerate vs r vs Filter fps

https://stackoverflow.com/questions/51143100/framerate-vs-r-vs-filter-fps

Using ffmpeg to change framerate

https://stackoverflow.com/questions/45462731/using-ffmpeg-to-change-framerate/45465730

using -hwaccel nvdec produces 'No decoder surfaces left' with interlaced input and 3 or more b-frames

https://trac.ffmpeg.org/ticket/7562

