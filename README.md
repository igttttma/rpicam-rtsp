# 树莓派 5 摄像头 RTSP 推流（MediaMTX + rpicam-vid + ffmpeg）

本文记录在 Raspberry Pi 5（Ubuntu 24.04 LTS）环境下，使用 MediaMTX 搭建 RTSP 服务，并通过 `rpicam-vid` + `ffmpeg` 向服务推流的完整步骤。

## 环境要求

- 硬件：Raspberry Pi 5
- 系统：Ubuntu 24.04 LTS（aarch64）
- 已安装并可用的 `rpicam-apps`（可用 `rpicam-vid --list-cameras` 查看摄像头）
- 已安装 `ffmpeg`：`sudo apt install -y ffmpeg`
- 已下载 MediaMTX 压缩包（示例：`/home/pi/mediamtx_v1.15.5_linux_arm64.tar.gz`）

## 安装 MediaMTX

1. 解压到指定目录：
   ```bash
   mkdir -p /home/pi/mediamtx
   tar -xzf /home/pi/mediamtx_v1.15.5_linux_arm64.tar.gz -C /home/pi/mediamtx
   ls -la /home/pi/mediamtx
   ```
   解压后目录包含：
   - `mediamtx`（可执行文件）
   - `mediamtx.yml`（默认配置）

## 以 systemd 管理 MediaMTX（开机自启）

1. 创建 systemd 单元文件 `/etc/systemd/system/mediamtx.service`：
   ```ini
   [Unit]
   Description=MediaMTX (RTSP/RTMP/HLS/WebRTC) Server
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   User=pi
   Group=pi
   WorkingDirectory=/home/pi/mediamtx
   ExecStart=/home/pi/mediamtx/mediamtx
   Restart=on-failure
   RestartSec=3
   LimitNOFILE=65536

   [Install]
   WantedBy=multi-user.target
   ```

2. 重新加载并启用服务：
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable mediamtx
   sudo systemctl start mediamtx
   ```

3. 查看状态与端口：
   ```bash
   systemctl --no-pager -l status mediamtx
   ss -ltnp | grep 8554
   ```
   正常情况下可见 `mediamtx` 进程监听 `8554` 端口，同时还会打开 `1935`（RTMP）、`8888`（HLS）、`8889`（WebRTC）等端口。

> 若此前使用过 `nohup ./mediamtx &` 启动，请先停止该旧实例以避免端口被占用：
> ```bash
> sudo pkill -x mediamtx
> ```

## 摄像头推流到 MediaMTX（RTSP）

1. 建议先确认摄像头列表：
   ```bash
   rpicam-vid --list-cameras
   ```

2. 通过 `rpicam-vid` 采集并使用 `ffmpeg` 推送到 RTSP（路径名示例：`cam`）：
   ```bash
   # 1080p@30，低延迟参数，使用 TCP 传输
   sudo rpicam-vid -t 0 --nopreview --codec libav --libav-format h264 \
     --low-latency --width 1920 --height 1080 --framerate 30 -o - \
     | ffmpeg -hide_banner -loglevel warning -fflags nobuffer -an -f h264 -i - \
       -c:v copy -f rtsp -rtsp_transport tcp -muxdelay 0.1 -muxpreload 0.1 \
       rtsp://127.0.0.1:8554/cam
   ```

   说明：
   - `--nopreview`：不创建预览窗口（避免在无图形环境下报错）
   - `sudo`：在 Ubuntu 24.04 下，`rpicam-vid` 访问 DMA 堆可能需要 root 权限
   - `--codec libav --libav-format h264`：强制输出标准 H.264 原始流，利于后续推送
   - `ffmpeg` 侧使用 `-c:v copy` 直接封装为 RTSP 输出，`-rtsp_transport tcp` 减少丢包

## 推流命令作为 systemd 服务（1080p 开机自启）

1. 创建 `/etc/systemd/system/rpicam-rtsp.service`：
   ```ini
   [Unit]
   Description=RPi Cam RTSP push (720p ultra-low-latency)
   After=network-online.target mediamtx.service
   Wants=network-online.target mediamtx.service

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/home/pi
   ExecStart=/bin/bash -lc "rpicam-vid -t 0 --nopreview --codec libav --libav-format h264 --low-latency --intra 1 --inline --width 1280 --height 720 --framerate 25 --libav-video-codec h264_v4l2m2m -o - | ffmpeg -hide_banner -loglevel warning -fflags nobuffer -flags low_delay -fflags flush_packets -an -f h264 -i - -c:v copy -f rtsp -rtsp_transport udp -muxdelay 0 -muxpreload 0 rtsp://127.0.0.1:8554/cam"
   Restart=always
   RestartSec=1
   LimitNOFILE=65536

   [Install]
   WantedBy=multi-user.target
   ```

2. 启用并启动：
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable rpicam-rtsp
   sudo systemctl start rpicam-rtsp
   ```

3. 验证：
   ```bash
   systemctl --no-pager -l status rpicam-rtsp
   ffprobe -hide_banner -rtsp_transport tcp rtsp://<树莓派IP>:8554/cam
   ```

> 该服务与 `mediamtx.service` 建立依赖关系，确保开机后自动启动推流。若需要修改分辨率或帧率，直接编辑单元文件中的 `ExecStart`。

### 超低延迟优化建议（最终配置）

- 编码侧：
  - `--low-latency` 启用低延迟编码路径。
  - `--intra 1` **（关键）** 将关键帧间隔设为 1，最大程度减少解码器等待时间，是实现亚秒级延迟的核心。这会增加带宽，但对延迟改善效果最显著。
  - `--inline` 确保 I 帧附带 SPS/PPS 头信息。
  - `--width 1280 --height 720 --framerate 25` 降低分辨率和帧率可显著减轻硬件编码压力。
  - `--libav-video-codec h264_v4l2m2m` 使用硬件编码加速。
- 封装与传输：
  - `ffmpeg` 使用 `-fflags nobuffer -flags low_delay -fflags flush_packets` 减少缓冲。
  - `-rtsp_transport udp` **（关键）** 在局域网等可靠网络下，使用 UDP 替代 TCP 可避免重传导致的延迟累积。
  - `-muxdelay 0 -muxpreload 0` 进一步减少复用器的缓冲。
- 客户端：
  - 优先使用 **WebRTC** 协议播放（访问 `http://<树莓派IP>:8889/cam`），它是实现最低延迟的首选。
  - 若使用 VLC 等 RTSP 客户端，务必将其内部的网络缓存（caching）时间调至最低（如 100ms）。

3. 在局域网客户端播放（VLC/FFplay/NVR 等）：
   ```
   rtsp://<树莓派IP>:8554/cam
   ```
   可用 `ffprobe` 验证：
   ```bash
   ffprobe -hide_banner -rtsp_transport tcp rtsp://<树莓派IP>:8554/cam
   ```

## HLS / WebRTC 访问（可选）

- HLS（浏览器访问）：
  ```
  http://<树莓派IP>:8888/cam/
  ```
- WebRTC（浏览器访问）：
  ```
  http://<树莓派IP>:8889/
  ```
  在页面中选择 `cam` 路径进行播放。

## 常见问题与排查

- `Could not open any dmaHeap device / failed to allocate capture buffers`：
  - 使用 `sudo rpicam-vid ...` 运行，或检查内核/驱动是否匹配。
- RTSP 连接失败或丢包：
  - 客户端强制使用 TCP（VLC 设置：RTSP over TCP），服务端推送已使用 `-rtsp_transport tcp`。
- 1080p 推流：
  - 将命令中的 `--width/--height` 调整为 `1920/1080`；若出现资源不足或报错，可降低分辨率或帧率。

## 服务管理命令

```bash
sudo systemctl status mediamtx
sudo systemctl restart mediamtx
sudo systemctl stop mediamtx
sudo systemctl disable mediamtx
journalctl -u mediamtx -f
```

## 目录与文件

- MediaMTX 安装目录：`/home/pi/mediamtx`
- 配置文件：`/home/pi/mediamtx/mediamtx.yml`
- systemd 单元文件：`/etc/systemd/system/mediamtx.service`
- 本说明文档：`/home/pi/rpicam-rtsp/README.md`
