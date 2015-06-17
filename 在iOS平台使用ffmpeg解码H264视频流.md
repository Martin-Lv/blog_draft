对于视频文件和rtsp之类的主流视频传输协议，ffmpeg提供avformat_open_input接口，直接将文件路径或URL传入即可打开。读取视频数据、解码器初始参数设置等，都可以通过调用API来完成。

但是对于h264流，没有任何封装格式，也就无法使用libavformat。所以许多工作需要自己手工完成。

这里的h264流指AnnexB，也就是每个nal unit以起始码00 00 00 01 或 00 00 01开始的格式。关于h264码流格式，可以参考[这篇文章](http://www.szatmary.org/blog/25
)。

首先是手动设定AVCodec和AVCodecContext：

```
AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
AVCodecContext *codecCtx = avcodec_alloc_context3(codec);
avcodec_open2(codecCtx, codec, nil);
```

在AVCodecContext中会保存很多解码需要的信息，比如视频的长和宽，但是现在我们还不知道。

这些信息存储在h264流的SPS（序列参数集）和PPS（图像参数集）中。

对于每个nal unit，起始码后面第一个字节的后5位，代表这个nal unit的类型。7代表SPS，8代表PPS。一般在SPS和PPS后面的是IDR帧，无需前面帧的信息就可以解码，用5来代表。

检测nal unit类型的方法：

```
- (int)typeOfNalu:(NSData *)data
{
	char first = *(char *)[data bytes];
	return first & 0x1f;
}
```

264解码器在解码SPS和PPS的时候会提取出视频的信息，保存在AVCodecContext中。但是只把SPS和PPS传递进去是不行的，需要把后面的IDR帧一起传给解码器，才能够正确解码。

可以写一个简单的检测，如果接收到SPS，就把后面的PPS和IDR帧都接收过来，然后一起传给解码器。

初始化一个AVPacket和AVFrame，然后把SPS、PPS、IDR帧连在一起的数据块传给AVPacket的data指针，再进行解码。

我们假设包含SPS、PPS、IDR帧的数据块保存在videoData中，长度为len。

```
char *videoData;
int len;
AVFrame *frame = av_frame_alloc();
AVPacket packet;
av_new_packet(&packet, len);
memcpy(packet.data, videoData, len);
int ret, got_picture;
ret = avcodec_decode_video2(codecCtx, frame, &got_picture, &packet);
if (ret > 0){
	if(got_picture){
	//进行下一步的处理
	}
}

```


这样就可以顺利解码h264流了，解码出的数据保存在AVFrame中。

我写了一个Objective-C类用来执行接收视频流、解码、播放一系列步骤。

视频数据的接收采用socket直接接收，使用了开源项目[CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket)。

就像项目名称中指明的，这是一个异步socket类。读写socket的动作会在一个单独的dispatch queue中执行，执行完毕后对应的delegate方法会自动调用，在其中进行进一步的处理。

读取h264流使用了GCDAsyncSocket 的
`- (void)readDataToData:(NSData *)data withTimeout:(NSTimeInterval)timeout tag:(long)tag`
方法，也就是当读到和data中的字节一致的内容时就停止读取，并调用delegate方法。传入的data参数是 00 00 01 三个字节。这样每次读入的nalu开始是没有start code的，而最后面有下一个nalu的start code。因此每次读取之后都会把末尾的start code 暂存，然后把主体接到上一次暂存的start code之后，构成完整的nalu。

videoPlayer.h:

```
//videoPlayer.h
#import <Foundation/Foundation.h>

@interface videoPlayer : NSObject
	
- (void)startup;
- (void)shutdown;
@end
```

videoPlayer.m:

```

//videoPlayer.m
	
	
#import "videoPlayer.h"
#import "GCDAsyncSocket.h"
	
#import "libavcodec/avcodec.h"
#import "libswscale/swscale.h"
	
const int Header = 101;
const int Data = 102;
	
@interface videoPlayer () <GCDAsyncSocketDelegate>
{
    GCDAsyncSocket *socket;
    NSData *startcodeData;
    NSData *lastStartCode;
    
    //ffmpeg
    AVFrame *frame;
    AVPicture picture;
    AVCodec *codec;
    AVCodecContext *codecCtx;
    AVPacket packet;
    struct SwsContext *img_convert_ctx;
    
    NSMutableData *keyFrame;
    
    int outputWidth;
    int outputHeight;
}
@end
	
@implementation videoPlayer
	
- (id)init
{
    self = [super init];
    if (self) {
        avcodec_register_all();
        frame = av_frame_alloc();
        codec = avcodec_find_decoder(AV_CODEC_ID_H264);
        codecCtx = avcodec_alloc_context3(codec);
        int ret = avcodec_open2(codecCtx, codec, nil);
        if (ret != 0){
            NSLog(@"open codec failed :%d",ret);
        }
        
        socket = [[GCDAsyncSocket alloc]initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
        keyFrame = [[NSMutableData alloc]init];
	
        outputWidth = 320;
        outputHeight = 240;
        
        unsigned char startcode[] = {0,0,1};
        startcodeData = [NSData dataWithBytes:startcode length:3];
    }
    return self;
}
	
- (void)startup
{
    NSError *error = nil;
    [socket connectToHost:@"192.168.1.100"
                   onPort:9982
              withTimeout:-1
                    error:&error];
    NSLog(@"%@",error);
    if (!error) {
		[socket readDataToData:startcodeData withTimeout:-1 tag:0];
	}
}
	
- (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag
{
    [socket readDataToData:startcodeData withTimeout:-1 tag:Data];
    if(tag == Data){
        int type = [self typeOfNalu:data];
        if (type == 7 || type == 8 || type == 6 || type == 5) { //SPS PPS SEI IDR
            [keyFrame appendData:lastStartCode];
            [keyFrame appendBytes:[data bytes] length:[data length] - [self startCodeLenth:data]];
        }
        if (type == 5 || type == 1) {//IDR P frame
            if (type == 5) {
                int nalLen = (int)[keyFrame length];
                av_new_packet(&packet, nalLen);
                memcpy(packet.data, [keyFrame bytes], nalLen);
                keyFrame = [[NSMutableData alloc] init];//reset keyframe
            }else{
                NSMutableData *nalu = [[NSMutableData alloc]initWithData:lastStartCode];
                [nalu appendBytes:[data bytes] length:[data length] - [self startCodeLenth:data]];
                int nalLen = (int)[nalu length];
                av_new_packet(&packet, nalLen);
                memcpy(packet.data, [nalu bytes], nalLen);
            }
            
            int ret, got_picture;
            //NSLog(@"decode start");
            ret = avcodec_decode_video2(codecCtx, frame, &got_picture, &packet);
            //NSLog(@"decode finish");
            if (ret < 0) {
                NSLog(@"decode error");
                return;
            }
            if (!got_picture) {
                NSLog(@"didn't get picture");
                return;
            }
            static int sws_flags =  SWS_FAST_BILINEAR;
            //outputWidth = codecCtx->width;
            //outputHeight = codecCtx->height;
            if (!img_convert_ctx)
                img_convert_ctx = sws_getContext(codecCtx->width,
                                                 codecCtx->height,
                                                 codecCtx->pix_fmt,
                                                 outputWidth,
                                                 outputHeight,
                                                 PIX_FMT_YUV420P,
                                                 sws_flags, NULL, NULL, NULL);
            
            avpicture_alloc(&picture, PIX_FMT_YUV420P, outputWidth, outputHeight);
            ret = sws_scale(img_convert_ctx, (const uint8_t* const*)frame->data, frame->linesize, 0, frame->height, picture.data, picture.linesize);
	
            [self display];
            //NSLog(@"show frame finish");
            avpicture_free(&picture);
            av_free_packet(&packet);
        }
    }
    [self saveStartCode:data];
}
	
- (void)display
{
	
}
	
- (int)typeOfNalu:(NSData *)data
{
    char first = *(char *)[data bytes];
    return first & 0x1f;
}
	
- (int)startCodeLenth:(NSData *)data
{
    char temp = *((char *)[data bytes] + [data length] - 4);
    return temp == 0x00 ? 4 : 3;
}
	
- (void)saveStartCode:(NSData *)data
{
    int startCodeLen = [self startCodeLenth:data];
    NSRange startCodeRange = {[data length] - startCodeLen, startCodeLen};
    lastStartCode = [data subdataWithRange:startCodeRange];
}
	
- (void)shutdown
{
    if(socket)[socket disconnect];
}
	
- (void)dealloc
{
    // Free scaler
	if(img_convert_ctx)sws_freeContext(img_convert_ctx);
	
    // Free the YUV frame
    if(frame)av_frame_free(&frame);
    
    // Close the codec
    if (codecCtx) avcodec_close(codecCtx);
}
	
@end
	
```
在项目中播放解码出来的YUV视频使用了OPENGL，这里播放的部分就略去了。