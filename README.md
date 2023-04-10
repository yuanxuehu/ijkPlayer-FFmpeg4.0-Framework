# ijkPlayer-FFmpeg4.0-Framework
基于ijkPlayer(FFmpeg4.0)编译Framework成功过程梳理

编译依赖
ffmpeg版本：ff4.0--ijk0.8.8--20210426--001
openssl版本：OpenSSL_1_0_2u
编译成功日期：2021.7.1
 
附参考链接：
https://blog.csdn.net/baidu_31868779/article/details/107412113
https://hub.fastgit.org/SEEWO-inc/IJKMediaPlayer.git(镜像地址)
https://github.com/SEEWO-inc/IJKMediaPlayer.git

0x00 获取ijkplayer源码
#由于github被墙，解决方案是把ffmpeg的github仓库导入到码云gitee
git clone https://gitee.com/yuanxuehu2016/ijkplayer.git ijkplayer-ios
cd ijkplayer-ios

重要！！！
Ijkplayer的编译脚本一定要选tag是k0.8.8
git checkout -B latest k0.8.8


0x01 编译ijkplayer
ijkplayer的编译依赖的另外两个仓库
1.https://github.com/bilibili/FFmpeg
2.https://github.com/bilibili/openssl
 
编译步骤也非常简单，参考官方的编译指导即可
step1: 获取依赖的ffmpeg 和 openssl源码，配置支持的各种架构
#拉取ffmpeg 放入到extra/ffmpeg
./init-ios.sh        
#用于支持https协议, 拉取openssl 放入到extra/openssl中
./init-ios-openssl.sh
 
 
在init-ios.sh文件里，IJK_FFMPEG_COMMIT字段定义了依赖的ffmpeg源码git tag。
 例如：
IJK_FFMPEG_UPSTREAM=https://gitee.com/yuanxuehu2016/FFmpeg.git
IJK_FFMPEG_FORK=https://gitee.com/yuanxuehu2016/FFmpeg.git
IJK_FFMPEG_COMMIT=ff4.0--ijk0.8.8--20210426--001
IJK_FFMPEG_LOCAL_REPO=extra/ffmpeg
 
IJK_GASP_UPSTREAM=https://gitee.com/yuanxuehu2016/gas-preprocessor.git
 
 
在init-ios-openssl.sh文件里 ，IJK_OPENSSL_COMMIT字段定义了依赖的openssl源码git tag。
 例如：
IJK_OPENSSL_UPSTREAM=https://gitee.com/yuanxuehu2016/originalOpenssl
IJK_OPENSSL_FORK=https://gitee.com/yuanxuehu2016/openssl.git
IJK_OPENSSL_COMMIT=OpenSSL_1_0_2u
 
step2: 配置ffmpeg编译选项
在config目录下有三个配置文件，可以按需选择一种配置文件链接到module.sh
module-default.sh #更多的编解码器/格式，编译出的包体积会很大
module-lite-hevc.sh #较少的编解码器/格式(包括hevc)
module-lite.sh #较少的编解码器/格式(默认情况)，编译出的包体积较小
 
 选择module-lite.sh
cd config
rm module.sh
ln -s module-lite.sh module.sh
 
 
# 在模块文件config/module.sh中添加一行配置 以启用 openssl 组件 
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"' >> ../config/module.sh
 
# 在module-lite的基础上支持avi，rm, rmvb, 3gp，mpg, wmv的视频，增加的编译选项 
# 开启demuxer编译选项
(由于过长，放在了文档最后)
 

重点：
修改ffmpeg4.0需要的编译配置选项
在config/mudule.sh文件中
 
# 关闭bitstream filter（ffmpeg4.0）
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-bsf=eac3_core"
 
# 注释掉这两行
#export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-ffserver"
#export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-vda"

# 注释
#export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-libx264"
 
 
修改 ios/tools/do-compile-ffmpeg.sh和ios/tools/do-compile-openssl.sh中支持的最低版本
找到有关 version-min 小于8.0的都改成8.0
 
 
step3:  切换到ios目录编译ffmpeg和openssl
 
cd ios
 
# 清除build目录下的文件（包括ffmpeg、openssl、universal）
./compile-ffmpeg.sh clean
 
./compile-openssl.sh all
 
./compile-ffmpeg.sh all
 
 
 
 
 
step4: 编译ijkplaye framework
 
打开IJKMediaPlayer/IJKMediaPlayer.xcodeproj即可编译（或使用xcodebuild命令行编译）
 
IJKMediaPlayer默认有两个target：IJKMediaFramework 和 IJKMediaFrameworkWithSSL
 
这两个target一个是Dynamic Framework（IJKMediaFrameworkWithSSL），一个是Static Framework（IJKMediaFramework），支持的特性是一样的。
 
其中IJKMediaFrameworkWithSSL.framework在使用时需要Embed使用
 
Edit Scheme将debug改为release模式
先选模拟器build,再选真机build
模拟器编译架构选x86_64,真机选arm64

step5: 使用lipo合成arm64+x86_64 FAT framework
静态库
lipo -create Release-iphoneos/IJKMediaFramework.framework/IJKMediaFramework Release-iphonesimulator/IJKMediaFramework.framework/IJKMediaFramework -output IJKMediaFramework
 
动态库
lipo -create Release-iphoneos/IJKMediaFrameworkWithSSL.framework/IJKMediaFrameworkWithSSL
Release-iphonesimulator/IJKMediaFrameworkWithSSL.framework/IJKMediaFrameworkWithSSL -output IJKMediaFrameworkWithSSL
 
将生成的 IJKMediaFramework 替换掉 Release-iphoneos/IJKMediaFramework.framework 文件下的 IJKMediaFramework，就 OK 啦。
将生成的 IJKMediaFrameworkWithSSL 替换掉 Release-iphoneos/IJKMediaFrameworkWithSSL.framework 文件下的 IJKMediaFrameworkWithSSL，就 OK 啦。
 


Step6: 将生成的 framework 导入到所需工程中

1）直接将 IJKMediaFrameworkWithSSL.framework 拖入到工程中即可，注意记得勾选 Copy items if needed 和 对应的 target。

2）然后添加相关依赖

libc++.tbd ( 编译器选 gcc 的请导入 libstdc++.tbd )
libz.tbd
libbz2.tbd
AudioToolbox.framework
UIKit.framework
CoreGraphics.framework
AVFoundation.framework
CoreMedia.framework
CoreVideo.framework
MediaPlayer.framework
MobileCoreServices.framework
OpenGLES.framework
QuartzCore.framework
VideoToolbox.framework
3）在相应文件中导入 #import <IJKMediaFrameworkWithSSL/IJKMediaPlayer.h>
即可使用 IJKMediaPlayer 了

4）如果运行程序报 image not found 错误 ，需在工程对应 TARGETS —— General —— Embedded Binaries 中添加 IJKMediaFrameworkWithSSL.framework 即可.


 
 
 
 
 
附件
# 开启demuxer编译选项
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4v"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4a"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=wmv*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=asf*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=webm"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=caf"' >> ../config/module.sh
 
# # 开启decoder编译选项
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=h263*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=cook"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=rv*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=ac3*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp2*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp1*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=wmv*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=msmpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=alac"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=amr*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"' >> ../config/module.sh
 
#  开启parser编译选项
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=ac3"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=mpeg*"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=cook"' >> ../config/module.sh
echo 'export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=h263"' >> ../config/module.sh
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4v"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4a"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=asf*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=webm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=caf"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=h263*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=rv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=ac3*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp2*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp1*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=alac"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=amr*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=ac3"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=h263"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4v"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4a"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=asf*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=webm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=caf"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=h263*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=rv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=ac3*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp2*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp1*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=alac"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=amr*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=ac3"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=h263"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4v"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4a"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=asf*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=webm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=caf"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=h263*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=rv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=ac3*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp2*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp1*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=alac"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=amr*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=ac3"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=h263"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4v"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=m4a"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=avi"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=asf*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=webm"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=caf"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=h263*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=rv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=ac3*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp2*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mp1*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=wmv*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=msmpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=alac"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=amr*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=ac3"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=mpeg*"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=cook"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=h263"
 
