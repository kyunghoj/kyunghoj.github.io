---
title: Compiling ffmpeg with NVIDIA GPU Acceleration
categoris:
  - Blog
tags: [ffmpeg, nvidia, hwaccel, gpu]
---


> [!WARNING] 재생을 하지 않는 비디오 디코딩의 경우, 대부분의 최신 CPU 는 NVIDIA GPU를 사용했을 때와 크게 다르지 않은 성능을 보여준다고 한다. 특히나 av1 같은 비디오 디코더가 내장된 데스크탑 CPU라면 더더욱 그럴 것이다. `ffmpeg` 을 직접 빌드하는 것은 많은 문제를 발생시킬 가능성이 높으므로, 웬만하면 하지 않는 것을 추천한다.

`ffmpeg` 이 NVIDIA GPU 를 이용한 가속을 하게 컴파일하려면 `nvcc` 를 필요로 하는데, `nvcc`는 [CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit) 에 포함되어 있다.

먼저 리눅스 커널 헤더가 있는지 확인하자. (시스템 리부트 후, 리눅스 커널이 업데이트 되었을 때에도 설치가 필요하다.)

```
sudo apt install linux-headers-`uname -r` dkms
```

이전에 이미 NVIDIA Driver를 설치했다면 버전이 맞지 않을 수 있다. 지워버린다. (따라서 처음부터 그냥 CUDA Toolkit 을 설치하는 것이 나을 것이다.)

```
sudo nvidia-uninstall
```

설치파일을 다운로드하고 설치한다:

```
wget https://developer.download.nvidia.com/compute/cuda/12.8.0/local_installers/cuda_12.8.0_570.86.10_linux.run
sudo sh cuda_12.8.0_570.86.10_linux.run
```

설치하는 시점에 따라 버전이 다를 수 있다. 설치파일 용량이 5GB 쯤 되므로, 파일시스템에 충분한 공간을 확보한 후에 진행하자.

설치 결과, 아래와 같은 출력이 나와야 한다.

```
$ sudo ./cuda_12.8.0_570.86.10_linux.run 
===========
= Summary =
===========

Driver:   Installed
Toolkit:  Installed in /usr/local/cuda-12.8/

Please make sure that
 -   PATH includes /usr/local/cuda-12.8/bin
  -   LD_LIBRARY_PATH includes /usr/local/cuda-12.8/lib64, or, add /usr/local/cuda-12.8/lib64 to /etc/ld.so.conf and run ldconfig as root

  To uninstall the CUDA Toolkit, run cuda-uninstaller in /usr/local/cuda-12.8/bin
  To uninstall the NVIDIA Driver, run nvidia-uninstall
  Logfile is /var/log/cuda-installer.log
  ```

## Compiling `ffmpeg`

### `ffnvcodec`

- Clone `ffnvcodec`

```
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
```

- Install `ffnvcodec`

```
cd nv-codec-headers && sudo make install && cd –
```

- Clone `ffmpeg`

```
git clone https://git.ffmpeg.org/ffmpeg.git ffmpeg/
```

- Install necessary packages

```
sudo apt-get install build-essential yasm cmake libtool libc6 libc6-dev unzip wget libnuma1 libnuma-dev pkg-config libx264-dev libx265-dev libaom-dev libdav1d-dev libsvtav1-dev
```

- Configure

```
./configure --enable-nonfree --enable-cuda-nvcc --enable-libnpp --extra-cflags=-I/usr/local/cuda/include --extra-ldflags=-L/usr/local/cuda/lib64 --disable-static --enable-shared --enable-gpl --enable-libx264 --enable-libx265 --enable-libaom --enable-libdav1d --enable-libsvtav1
```

- Compile and Install the libraries

```
make -j 8 & sudo make install
```

동적 라이브러리 로드가 안 되어 아래와 같은 에러가 발생할 수 있다.

```
ffmpeg: error while loading shared libraries: libavdevice.so.53: cannot open shared object file: No such file or directory
```


`/etc/ld.so.conf` 를 열어 `/usr/local/lib` 을 추가하고 `sudo ldconfig` 라는 명령을 실행한다. 

## References

1. [https://pyav.org/docs/9.0.2/overview/caveats.html?highlight=hwaccel](https://pyav.org/docs/9.0.2/overview/caveats.html?highlight=hwaccel)
2. [Using FFmpeg with NVIDIA GPU Hardware Acceleration](https://docs.nvidia.com/video-technologies/video-codec-sdk/11.1/ffmpeg-with-nvidia-gpu/index.html)



