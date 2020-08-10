```Object-C

//
//  H264HwDecoder.h
//  runtimess
//
//  Created by LiuGen on 2020/8/8.
//  Copyright © 2020 ProDrone. All rights reserved.
//

//
// H264HwDecoder.h
// IOTCamera
//
// Created by lzj<lizhijian_21@163.com> on 2017/2/18.
// Copyright (c) 2017 LZJ. All rights reserved.
//

#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <VideoToolbox/VideoToolbox.h>
#import <AVFoundation/AVSampleBufferDisplayLayer.h>

typedef enum : NSUInteger {
  H264HWDataType_Image = 0,
  H264HWDataType_Pixel,
  H264HWDataType_Layer,
} H264HWDataType;

@interface H264HwDecoder : NSObject

@property (nonatomic,assign) H264HWDataType showType;  //显示类型
@property (nonatomic,strong) UIImage *image;      //解码成RGB数据时的IMG
@property (nonatomic,assign) CVPixelBufferRef pixelBuffer;  //解码成YUV数据时的解码BUF
@property (nonatomic,strong) AVSampleBufferDisplayLayer *displayLayer; //显示图层

@property (nonatomic,assign) BOOL isNeedPerfectImg;  //是否读取完整UIImage图形(showType为0时才有效)

- (instancetype)init;

-(void)putData:(NSData *)data;

-(void)startDecode;

/**
H264视频流解码
@param videoData 视频帧数据
@param videoSize 视频帧大小
@return 视图的宽高(width, height)，当为接收为AVSampleBufferDisplayLayer时返回接口是无效的
*/
- (CGSize)decodeH264VideoData:(uint8_t *)videoData videoSize:(NSInteger)videoSize;

/**
释放解码器
*/
- (void)releaseH264HwDecoder;

/**
视频截图
@return IMG
*/
- (UIImage *)snapshot;

@end


```

```Object-C
//
//  H264HwDecoder.m
//  runtimess
//
//  Created by LiuGen on 2020/8/8.
//  Copyright © 2020 ProDrone. All rights reserved.
//
//
//  H264HwDecoder.m
//  IOTCamera
//
//  Created by lzj<lizhijian_21@163.com> on 2017/2/18.
//  Copyright (c) 2017 LZJ. All rights reserved.
//
 
#import "H264HwDecoder.h"
 
#ifndef FreeCharP
#define FreeCharP(p) if (p) {free(p); p = NULL;}
#endif
 
typedef enum : NSUInteger {
    HWVideoFrameType_UNKNOWN = 0,
    HWVideoFrameType_I,
    HWVideoFrameType_P,
    HWVideoFrameType_B,
    HWVideoFrameType_SPS,
    HWVideoFrameType_PPS,
    HWVideoFrameType_SEI,
} HWVideoFrameType;
 
@interface H264HwDecoder ()
{
    VTDecompressionSessionRef mDeocderSession;
    CMVideoFormatDescriptionRef mDecoderFormatDescription;
    
    uint8_t *pSPS;
    uint8_t *pPPS;
    uint8_t *pSEI;
    NSInteger mSpsSize;
    NSInteger mPpsSize;
    NSInteger mSeiSize;
    
    NSInteger mINalCount;        //I帧起始码个数
    NSInteger mPBNalCount;       //P、B帧起始码个数
    NSInteger mINalIndex;       //I帧起始码开始位
    
    BOOL mIsNeedReinit;         //需要重置解码器
    
    NSMutableArray <NSData*>* packageData;
}
 
@end
 
static void didDecompress(void *decompressionOutputRefCon, void *sourceFrameRefCon, OSStatus status, VTDecodeInfoFlags infoFlags, CVImageBufferRef pixelBuffer, CMTime presentationTimeStamp, CMTime presentationDuration )
{
    CVPixelBufferRef *outputPixelBuffer = (CVPixelBufferRef *)sourceFrameRefCon;
    *outputPixelBuffer = CVPixelBufferRetain(pixelBuffer);
}
 
@implementation H264HwDecoder
 
- (instancetype)init
{
    if (self = [super init]) {
        pSPS = pPPS = pSEI = NULL;
        mSpsSize = mPpsSize = mSeiSize = 0;
        mINalCount = mPBNalCount = mINalIndex = 0;
        mIsNeedReinit = NO;
        
        _showType = H264HWDataType_Image;
        _isNeedPerfectImg = NO;
        _pixelBuffer = NULL;
        packageData = [NSMutableArray array];
    }
    
    return self;
}

- (void)dealloc
{
    [self releaseH264HwDecoder];
}
 
- (BOOL)initH264HwDecoder
{
    if (mDeocderSession) {
        return YES;
    }
    
    const uint8_t *const parameterSetPointers[2] = {pSPS,pPPS};
    const size_t parameterSetSizes[2] = {mSpsSize, mPpsSize};
    
    OSStatus status = CMVideoFormatDescriptionCreateFromH264ParameterSets(kCFAllocatorDefault, 2, parameterSetPointers, parameterSetSizes, 4, &mDecoderFormatDescription);
    
    if (status == noErr) {
        //      kCVPixelFormatType_420YpCbCr8Planar is YUV420
        //      kCVPixelFormatType_420YpCbCr8BiPlanarFullRange is NV12
        //      kCVPixelFormatType_24RGB    //使用24位bitsPerPixel
        //      kCVPixelFormatType_32BGRA   //使用32位bitsPerPixel，kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst
    uint32_t pixelFormatType = kCVPixelFormatType_420YpCbCr8BiPlanarFullRange;  //NV12
    if (self.showType == H264HWDataType_Pixel) {
        pixelFormatType = kCVPixelFormatType_420YpCbCr8Planar;
    }
    const void *keys[] = { kCVPixelBufferPixelFormatTypeKey };
    const void *values[] = { CFNumberCreate(NULL, kCFNumberSInt32Type, &pixelFormatType) };
    CFDictionaryRef attrs = CFDictionaryCreate(NULL, keys, values, 1, NULL, NULL);
    
    VTDecompressionOutputCallbackRecord callBackRecord;
    callBackRecord.decompressionOutputCallback = didDecompress;
    callBackRecord.decompressionOutputRefCon = NULL;
    
    status = VTDecompressionSessionCreate(kCFAllocatorDefault,
                                          mDecoderFormatDescription,
                                          NULL, attrs,
                                          &callBackRecord,
                                          &mDeocderSession);
    CFRelease(attrs);
        NSLog(@"Init H264 hardware decoder success");
    } else {
        NSLog([NSString stringWithFormat:@"Init H264 hardware decoder fail: %d", (int)status]);
        return NO;
    }
    
    return YES;
}
 
- (void)removeH264HwDecoder
{
    if(mDeocderSession) {
        VTDecompressionSessionInvalidate(mDeocderSession);
        CFRelease(mDeocderSession);
        mDeocderSession = NULL;
    }
    
    if(mDecoderFormatDescription) {
        CFRelease(mDecoderFormatDescription);
        mDecoderFormatDescription = NULL;
    }
}
 
- (void)releaseH264HwDecoder
{
    [self removeH264HwDecoder];
    [self releaseSliceInfo];
    
    if (_pixelBuffer) {
        CVPixelBufferRelease(_pixelBuffer);
        _pixelBuffer = NULL;
    }
}
 
- (void)releaseSliceInfo
{
    FreeCharP(pSPS);
    FreeCharP(pPPS);
    FreeCharP(pSEI);
    
    mSpsSize = 0;
    mPpsSize = 0;
    mSeiSize = 0;
}
 
-(void)putData:(NSData *)data
{
    [packageData addObject:data];
}

-(void)startDecode{
    
    NSBlockOperation *blockOPeration = [NSBlockOperation blockOperationWithBlock:^{
        
        NSLog(@"=lastTime=%d",[NSThread isMainThread]);
        NSTimeInterval lastTime = 0;
        while(packageData.count > 0){
            
            NSTimeInterval currentTime = [NSDate date].timeIntervalSince1970*1000;
            NSTimeInterval betweenTime = currentTime - lastTime;
            
            NSTimeInterval sleepTime = 5;
            if(betweenTime > 25){
                sleepTime = 5;
            }else if (betweenTime > 10 && betweenTime <= 25){
                sleepTime = 20;
            }
            NSData *data = packageData.firstObject;
            //[packageData removeObjectAtIndex:0];
            [self decodeH264VideoData:(uint8_t*)[data bytes] videoSize:data.length];
            
            [NSThread sleepForTimeInterval:sleepTime/1000];
            lastTime = currentTime;
            
        }
    }];
    
    NSOperationQueue *queue = [NSOperationQueue currentQueue];
    [queue addOperation:blockOPeration];

}

//将视频数据封装成CMSampleBufferRef进行解码
- (CVPixelBufferRef)decode:(uint8_t *)videoBuffer videoSize:(NSInteger)videoBufferSize
{
    CVPixelBufferRef outputPixelBuffer = NULL;
    CMBlockBufferRef blockBuffer = NULL;
    OSStatus status  = CMBlockBufferCreateWithMemoryBlock(kCFAllocatorDefault, (void *)videoBuffer, videoBufferSize, kCFAllocatorNull, NULL, 0, videoBufferSize, 0, &blockBuffer);
    if (status == kCMBlockBufferNoErr) {
        CMSampleBufferRef sampleBuffer = NULL;
        const size_t sampleSizeArray[] = { videoBufferSize };
        status = CMSampleBufferCreateReady(kCFAllocatorDefault, blockBuffer, mDecoderFormatDescription , 1, 0, NULL, 1, sampleSizeArray, &sampleBuffer);
        
        if (status == kCMBlockBufferNoErr && sampleBuffer) {
            if (self.showType == H264HWDataType_Layer && _displayLayer) {
                CFArrayRef attachments = CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, YES);
                CFMutableDictionaryRef dict = (CFMutableDictionaryRef)CFArrayGetValueAtIndex(attachments, 0);
                CFDictionarySetValue(dict, kCMSampleAttachmentKey_DisplayImmediately, kCFBooleanTrue);
                if ([self.displayLayer isReadyForMoreMediaData]) {
                    __weak typeof(self) weakSelf = self;
                    dispatch_sync(dispatch_get_main_queue(),^{
                    
                        [weakSelf.displayLayer enqueueSampleBuffer:sampleBuffer];
                    });
                }
                
                CFRelease(sampleBuffer);
            } else {
                VTDecodeFrameFlags flags = 0;
                VTDecodeInfoFlags flagOut = 0;
                OSStatus decodeStatus = VTDecompressionSessionDecodeFrame(mDeocderSession, sampleBuffer, flags, &outputPixelBuffer, &flagOut);
                CFRelease(sampleBuffer);
                if (decodeStatus == kVTVideoDecoderMalfunctionErr) {
                    NSLog(@"Decode failed status: kVTVideoDecoderMalfunctionErr");
                    CVPixelBufferRelease(outputPixelBuffer);
                    outputPixelBuffer = NULL;
                } else if(decodeStatus == kVTInvalidSessionErr) {
                    NSLog(@"Invalid session, reset decoder session");
                    [self removeH264HwDecoder];
                } else if(decodeStatus == kVTVideoDecoderBadDataErr) {
                    NSLog(@"%@", [NSString stringWithFormat:@"Decode failed status=%d(Bad data)", (int)decodeStatus]);
                } else if(decodeStatus != noErr) {
                    NSLog(@"%@", [NSString stringWithFormat:@"Decode failed status=%d", (int)decodeStatus]);
                }
            }
        }
        
        CFRelease(blockBuffer);
    }
    
    return outputPixelBuffer;
}
 
- (CGSize)decodeH264VideoData:(uint8_t *)videoData videoSize:(NSInteger)videoSize
{
    CGSize imageSize = CGSizeMake(0, 0);
    if (videoData && videoSize > 0) {
        HWVideoFrameType frameFlag = [self analyticalData:videoData size:videoSize];
        if (mIsNeedReinit) {
            mIsNeedReinit = NO;
            [self removeH264HwDecoder];
        }
        
        if (pSPS && pPPS && (frameFlag == HWVideoFrameType_I || frameFlag == HWVideoFrameType_P || frameFlag == HWVideoFrameType_B)) {
            uint8_t *buffer = NULL;
            if (frameFlag == HWVideoFrameType_I) {
                int nalExtra = (mINalCount==3?1:0);      //如果是3位的起始码，转为大端时需要增加1位
                videoSize -= mINalIndex;
                buffer = (uint8_t *)malloc(videoSize + nalExtra);
                memcpy(buffer + nalExtra, videoData + mINalIndex, videoSize);
                videoSize += nalExtra;
            } else {
                int nalExtra = (mPBNalCount==3?1:0);
                buffer = (uint8_t *)malloc(videoSize + nalExtra);
                memcpy(buffer + nalExtra, videoData, videoSize);
                videoSize += nalExtra;
            }
            
            uint32_t nalSize = (uint32_t)(videoSize - 4);
            uint32_t *pNalSize = (uint32_t *)buffer;
            *pNalSize = CFSwapInt32HostToBig(nalSize);
            
            CVPixelBufferRef pixelBuffer = NULL;
            if ([self initH264HwDecoder]) {
                pixelBuffer = [self decode:buffer videoSize:videoSize];
                
                if(pixelBuffer) {
                    NSInteger width = CVPixelBufferGetWidth(pixelBuffer);
                    NSInteger height = CVPixelBufferGetHeight(pixelBuffer);
                    imageSize = CGSizeMake(width, height);
                    
                    if (self.showType == H264HWDataType_Pixel) {
                        if (_pixelBuffer) {
                            CVPixelBufferRelease(_pixelBuffer);
                        }
                        self.pixelBuffer = CVPixelBufferRetain(pixelBuffer);
                    } else {
                        if (frameFlag == HWVideoFrameType_B) {  //若B帧未进行乱序解码，顺序播放，则在此需要去除，否则解码图形则是灰色。
                            size_t planeCount = CVPixelBufferGetPlaneCount(pixelBuffer);
                            if (planeCount >= 2 && planeCount <= 3) {
                                CVPixelBufferLockBaseAddress(pixelBuffer, 0);
                                u_char *yDestPlane = (u_char *)CVPixelBufferGetBaseAddressOfPlane(pixelBuffer, 0);
                                if (planeCount == 2) {
                                    u_char *uvDestPlane = (u_char *)CVPixelBufferGetBaseAddressOfPlane(pixelBuffer, 1);
                                    if (yDestPlane[0] == 0x80 && uvDestPlane[0] == 0x80 && uvDestPlane[1] == 0x80) {
                                        frameFlag = HWVideoFrameType_UNKNOWN;
                                        NSLog(@"Video YUV data parse error: Y=%02x U=%02x V=%02x", yDestPlane[0], uvDestPlane[0], uvDestPlane[1]);
                                    }
                                } else if (planeCount == 3) {
                                    u_char *uDestPlane = (u_char *)CVPixelBufferGetBaseAddressOfPlane(pixelBuffer, 1);
                                    u_char *vDestPlane = (u_char *)CVPixelBufferGetBaseAddressOfPlane(pixelBuffer, 2);
                                    if (yDestPlane[0] == 0x80 && uDestPlane[0] == 0x80 && vDestPlane[0] == 0x80) {
                                        frameFlag = HWVideoFrameType_UNKNOWN;
                                        NSLog(@"Video YUV data parse error: Y=%02x U=%02x V=%02x", yDestPlane[0], uDestPlane[0], vDestPlane[0]);
                                    }
                                }
                                CVPixelBufferUnlockBaseAddress(pixelBuffer, 0);
                            }
                        }
                        
                        if (frameFlag != HWVideoFrameType_UNKNOWN) {
                            self.image = [self pixelBufferToImage:pixelBuffer];
                        }
                    }
                    
                    CVPixelBufferRelease(pixelBuffer);
                }
            }
            
            FreeCharP(buffer);
        }
    }
    
    return imageSize;
}
 
- (UIImage *)pixelBufferToImage:(CVPixelBufferRef)pixelBuffer
{
    UIImage *image = nil;
    if (!self.isNeedPerfectImg) {
        //第1种绘制（可直接显示，不可保存为文件(无效缺少图像描述参数)）
        CIImage *ciImage = [CIImage imageWithCVPixelBuffer:pixelBuffer];
        image = [UIImage imageWithCIImage:ciImage];
    } else {
        //第2种绘制（可直接显示，可直接保存为文件，相对第一种性能消耗略大）
    CIImage *ciImage = [CIImage imageWithCVPixelBuffer:pixelBuffer];
    CIContext *temporaryContext = [CIContext contextWithOptions:nil];
    CGImageRef videoImage = [temporaryContext createCGImage:ciImage fromRect:CGRectMake(0, 0, CVPixelBufferGetWidth(pixelBuffer), CVPixelBufferGetHeight(pixelBuffer))];
    image = [[UIImage alloc] initWithCGImage:videoImage];
    CGImageRelease(videoImage);
    }
    
    return image;
}
 
- (UIImage *)snapshot
{
    UIImage *img = nil;
    if (self.displayLayer) {
        UIGraphicsBeginImageContext(self.displayLayer.bounds.size);
        [self.displayLayer renderInContext:UIGraphicsGetCurrentContext()];
        img = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
    } else {
        if (self.showType == H264HWDataType_Pixel) {
            if (self.pixelBuffer) {
                img = [self pixelBufferToImage:self.pixelBuffer];
            }
        } else {
            img = self.image;
        }
        
        if (!self.isNeedPerfectImg) {
            UIGraphicsBeginImageContext(CGSizeMake(img.size.width, img.size.height));
            [img drawInRect:CGRectMake(0, 0, img.size.width, img.size.height)];
            img = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
        }
    }
    
    return img;
}
 
 
//从起始位开始查询SPS、PPS、SEI、I、B、P帧起始码，遇到I、P、B帧则退出
//存在多种情况：
//1、起始码是0x0 0x0 0x0 0x01 或 0x0 0x0 0x1
//2、每个SPS、PPS、SEI、I、B、P帧为单独的Slice
//3、I帧中包含SPS、PPS、I数据Slice
//4、I帧中包含第3点的数据之外还包含SEI，顺序：SPS、PPS、SEI、I
//5、起始位是AVCC协议格式的大端数据(不支持多Slice的视频帧)
- (HWVideoFrameType)analyticalData:(const uint8_t *)buffer size:(NSInteger)size
{
    NSInteger preIndex = 0;
    HWVideoFrameType preFrameType = HWVideoFrameType_UNKNOWN;
    HWVideoFrameType curFrameType = HWVideoFrameType_UNKNOWN;
    for (int i=0; i<size && i < 300; i++) {       //一般第四种情况下的帧起始信息不会超过(32+256+12)位，可适当增大，为了不循环整个帧片数据
        int nalSize = [self getNALHeaderLen:(buffer + i) size:size-i];
        if(nalSize == 4){
            
        }
        if (nalSize == 0 && i == 0) {   //当每个Slice起始位开始若使用AVCC协议则判断帧大小是否一致
            uint32_t *pNalSize = (uint32_t *)(buffer);
            uint32_t videoSize = CFSwapInt32BigToHost(*pNalSize);    //大端模式转为系统端模式
            if (videoSize == size - 4) {     //是大端模式(AVCC)
                nalSize = 4;
            }
        }
    
        if (nalSize && i + nalSize + 1 < size) {
            int sliceType = buffer[i + nalSize] & 0x1F;
            NSLog(@"============ i= %d",i);
            NSLog(@"=====sliceType======sliceType= %d",sliceType);
            if (sliceType == 0x1) {
                mPBNalCount = nalSize;
                if (buffer[i + nalSize] == 0x1) {   //B帧
                    curFrameType = HWVideoFrameType_B;
                } else {    //P帧
                    curFrameType = HWVideoFrameType_P;
                }
                //i += nalSize;
                break;
            } else if (sliceType == 0x5) {     //IDR(I帧)
                if (preFrameType == HWVideoFrameType_PPS) {
                    mIsNeedReinit = [self getSliceInfo:buffer slice:&pPPS size:&mPpsSize start:preIndex end:i];
                } else if (preFrameType == HWVideoFrameType_SEI)  {
                    [self getSliceInfo:buffer slice:&pSEI size:&mSeiSize start:preIndex end:i];
                }
                
                mINalCount = nalSize;
                mINalIndex = i;
                //i += nalSize;
                curFrameType = HWVideoFrameType_I;
                goto Goto_Exit;
            } else if (sliceType == 0x7) {      //SPS
                preFrameType = HWVideoFrameType_SPS;
                preIndex = i + nalSize;
                i += nalSize;
            } else if (sliceType == 0x8) {      //PPS
                if (preFrameType == HWVideoFrameType_SPS) {
                    mIsNeedReinit = [self getSliceInfo:buffer slice:&pSPS size:&mSpsSize start:preIndex end:i];
                }
                
                preFrameType = HWVideoFrameType_PPS;
                preIndex = i + nalSize;
                i += nalSize;
            } else if (sliceType == 0x6) {      //SEI
                if (preFrameType == HWVideoFrameType_PPS) {
                    mIsNeedReinit = [self getSliceInfo:buffer slice:&pPPS size:&mPpsSize start:preIndex end:i];
                }
                
                preFrameType = HWVideoFrameType_SEI;
                preIndex = i + nalSize;
                i += nalSize;
            }
        }
    }
    
    //SPS、PPS、SEI为单独的Slice帧片
    if (curFrameType == HWVideoFrameType_UNKNOWN && preIndex != 0) {
        if (preFrameType == HWVideoFrameType_SPS) {
            mIsNeedReinit = [self getSliceInfo:buffer slice:&pSPS size:&mSpsSize start:preIndex end:size];
            curFrameType = HWVideoFrameType_SPS;
        } else if (preFrameType == HWVideoFrameType_PPS) {
             mIsNeedReinit = [self getSliceInfo:buffer slice:&pPPS size:&mPpsSize start:preIndex end:size];
            curFrameType = HWVideoFrameType_PPS;
        } else if (preFrameType == HWVideoFrameType_SEI)  {
            [self getSliceInfo:buffer slice:&pSEI size:&mSeiSize start:preIndex end:size];
            curFrameType = HWVideoFrameType_SEI;
        }
    }
    
Goto_Exit:
    return curFrameType;
}
 
//获取NAL的起始码长度是3还4
- (int)getNALHeaderLen:(const uint8_t *)buffer size:(NSInteger)size
{
    if (size >= 4 && buffer[0] == 0x0 && buffer[1] == 0x0 && buffer[2] == 0x0 && buffer[3] == 0x1) {
        return 4;
    } else if (size >= 3 && buffer[0] == 0x0 && buffer[1] == 0x0 && buffer[2] == 0x1) {
        return 3;
    }
    
    return 0;
}
 
//给SPS、PPS、SEI的Buf赋值，返回YES表示不同于之前的值
- (BOOL)getSliceInfo:(const uint8_t *)videoBuf slice:(uint8_t **)sliceBuf size:(NSInteger *)size start:(NSInteger)start end:(NSInteger)end
{
    BOOL isDif = NO;

    NSInteger len = end - start;
    uint8_t *tempBuf = (uint8_t *)(*sliceBuf);
    if (tempBuf) {
        if (len != *size || memcmp(tempBuf, videoBuf + start, len) != 0) {
           free(tempBuf);
           tempBuf = (uint8_t *)malloc(len);
           memcpy(tempBuf, videoBuf + start, len);
           
           *sliceBuf = tempBuf;
           *size = len;
           
           isDif = YES;
       }
    } else {
            tempBuf = (uint8_t *)malloc(len);
            memcpy(tempBuf, videoBuf + start, len);
        
            *sliceBuf = tempBuf;
            *size = len;
    }

    return isDif;
}
 
@end



```

