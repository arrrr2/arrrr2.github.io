---
title: FFmpeg 从入门到升天：1) Linux 下编译安装
date: 2023/12/22 14:45:00
cover: /2023/12/22/ffmpeg-install/neko.webp
coverWidth: 1600
coverHeight: 900
---
FFmpeg 好难用啊，但是又不能不用，我爱说实话。

## 说在前面
本篇内容参照 [https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu) 在 2023.12.22 的内容进行翻译（整理）。  
如有问题，可以过去看教程。  
虽然各个 linux 发行版都可以直接安装，但是考虑到可能缺少编码器，难以 debug 等种种原因，总会有人选择手动编译安装。
因此这篇教程就是这么用的。  

## 多线程
把 `make` 改成 `make -j$(nproc)`。  
不想使用全部的线程也可以指定，如 `make -j8`。

## 首先确定安装的位置
在官方说明中，所有中间文件放在用户文件夹的 ffmpeg_sources 和 ffmpg_build 下。这里为方便整理，专门创建一个文件夹用于整理ffmpeg的文件：  
`FFMPEG_PATH=$HOME/data/ffmpeg`  
如果想完全按照官方，将所有内容放在用户文件夹下（即 /home），那么只需要将上面改成  
`FFMPEG_PATH=$HOME`  
在 FFMPEG_PATH 下，先创建三个文件夹：    
`mkdir -p $FFMPEG_PATH/ffmpeg_sources $FFMPEG_PATH/ffmpeg_build ~/bin`

## 基础库的安装
执行：  
```bash
sudo apt-get update -qq && sudo apt-get -y install \
  autoconf \
  automake \
  build-essential \
  cmake \
  git-core \
  libass-dev \
  libfreetype6-dev \
  libgnutls28-dev \
  libmp3lame-dev \
  libsdl2-dev \
  libtool \
  libva-dev \
  libvdpau-dev \
  libvorbis-dev \
  libxcb1-dev \
  libxcb-shm0-dev \
  libxcb-xfixes0-dev \
  meson \
  ninja-build \
  pkg-config \
  texinfo \
  wget \
  yasm \
  zlib1g-dev
```
  
根据官方说明，Ubuntu 20.04 还需执行  
`sudo apt install libunistring-dev libaom-dev libdav1d-dev`  
如果 libdav1d-dev 无法安装，可以先放弃，之后编译安装。  

完事之后安装音视频相关的库。其中包括 x26x, vpx, aac, opus 和它们的依赖。  
`sudo apt-get install nasm libx264-dev libx265-dev libnuma-dev libvpx-dev libfdk-aac-dev libopus-dev`


## 没办法直接安装的库
三个av1的编解码器：aom-av1, dav1d-av1, svt-av1  
虽然它们的功能是一致的，但考虑到可能具有不一样的特性，建议全部选装。  
vmaf 是视频质量的评价标准之一，科研中可能会用到，建议选装。  
其中，从 Ubuntu 22.04LTS 起 dav1d 提供了官方包，可以使用`sudo apt-get install libdav1d-dev`，不需要手动安装。  

aom-av1：

```bash
cd FFMPEG_PATH=$HOME/data/ffmpeg_sources && \
git -C aom pull 2> /dev/null || git clone --depth 1 https://aomedia.googlesource.com/aom && \
mkdir -p aom_build && \
cd aom_build && \
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$FFMPEG_PATH/ffmpeg_build" -DENABLE_TESTS=OFF -DENABLE_NASM=on ../aom && \
PATH="$HOME/bin:$PATH" make && \
make install
```

dav1d-av1：

```shell
cd FFMPEG_PATH=$HOME/data/ffmpeg_sources && \
git -C SVT-AV1 pull 2> /dev/null || git clone https://gitlab.com/AOMediaCodec/SVT-AV1.git && \
mkdir -p SVT-AV1/build && \
cd SVT-AV1/build && \
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$FFMPEG_PATH/ffmpeg_build" -DCMAKE_BUILD_TYPE=Release -DBUILD_DEC=OFF -DBUILD_SHARED_LIBS=OFF .. && \
PATH="$HOME/bin:$PATH" make && \
make install
```

svt-av1:

```shell
cd FFMPEG_PATH=$HOME/data/ffmpeg_sources && \
git -C dav1d pull 2> /dev/null || git clone --depth 1 https://code.videolan.org/videolan/dav1d.git && \
mkdir -p dav1d/build && \
cd dav1d/build && \
meson setup -Denable_tools=false -Denable_tests=false --default-library=static .. --prefix "$FFMPEG_PATH/ffmpeg_build" --libdir="$FFMPEG_PATH/ffmpeg_build/lib" && \
ninja && \
ninja install
```


libvmaf: 

```shell
cd FFMPEG_PATH=$HOME/data/ffmpeg_sources && \
wget https://github.com/Netflix/vmaf/archive/v2.3.1.tar.gz && \
tar xvf v2.3.1.tar.gz && \
mkdir -p vmaf-2.3.1/libvmaf/build &&\
cd vmaf-2.3.1/libvmaf/build && \
meson setup -Denable_tests=false -Denable_docs=false --buildtype=release --default-library=static .. --prefix "$FFMPEG_PATH/ffmpeg_build" --bindir="$HOME/bin" --libdir="$FFMPEG_PATH/ffmpeg_build/lib" && \
ninja && \
ninja install
```


## 安装 FFMPEG
最后一步，愉快执行
```shell
cd FFMPEG_PATH=$HOME/data/ffmpeg_sources && \
wget -O ffmpeg-snapshot.tar.bz2 https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2 && \
tar xjvf ffmpeg-snapshot.tar.bz2 && \
cd ffmpeg && \
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$FFMPEG_PATH/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$FFMPEG_PATH/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$FFMPEG_PATH/ffmpeg_build/include" \
  --extra-ldflags="-L$FFMPEG_PATH/ffmpeg_build/lib" \
  --extra-libs="-lpthread -lm" \
  --ld="g++" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-gnutls \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libfreetype \
  --enable-libmp3lame \
  --enable-libopus \
  --enable-libsvtav1 \
  --enable-libdav1d \
  --enable-libvorbis \
  --enable-libvpx \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nonfree && \
PATH="$HOME/bin:$PATH" make && \
make install && \
hash -r
```

## 收尾
现在安装就已经全部完成，接下来执行
```shell
source ~/.profile
```
重启 shell ，输入 `ffmpeg` 即可运行。

最后执行
```shell
unset FFMPEG_PATH
```

释放目录变量。

## 可能遇到的问题
1 nasm libx264-dev libx265-dev libnuma-dev libvpx-dev libfdk-aac-dev libopus-dev 我都想手动编译。  
去 [https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu) 查看教程。  
  
2 为什么最后装在 $HOME/bin 文件夹下？  
参照官方说法，放在这个位置不影响系统文件、不和安装的包（包括 ffmpeg ）冲突，方便卸载或者移动，不需要root权限。  
  
3 如何卸载？  
删除 $HOME/bin 下有关 ffmpeg 的文件夹即可。    
`rm -rf ~/bin/{ffmpeg,ffprobe,ffplay,x264,x265,nasm}`  
如果想要执行之前的临时文件，可以执行  
`rm -rf $FFMPEG_PATH`  
$FFMPEG_PATH 为当时设置的目录，其实手动删除即可。  
   
4 我连不上外网  
两种办法可供参考：爬梯子，或者在本地把库下载了之后拷到 ffmpeg_resource 文件夹之下，再把开头为 git 或 wget 的那一行去掉。  