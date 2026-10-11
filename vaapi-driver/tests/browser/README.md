# 浏览器侧帧序验证

用于验证 Chrome 实际上屏的帧顺序与画面内容，覆盖 ffmpeg 测不到的路径。

## 为什么需要浏览器测试

`tests/regress_chrome_contract.sh` 用 `DMD_NO_MAP_WAIT=1` 模拟 Chrome 的
"EndPicture 返回即采样"契约，但它仍然是 ffmpeg 在驱动。有两件事只有真实
浏览器能覆盖：

- **MSE 分段喂流**：在线视频走 `appendBuffer`，时间轴基准与直读 `src` 不同
- **实际上屏内容**：Chrome 的软件 DPB 决定哪个 surface 何时上屏

## 判据

`requestVideoFrameCallback` 给出每次上屏的 `mediaTime`，据此算出帧号；
同时用 canvas 抓像素算签名，与软解基线比对，得到"画面里实际是第几帧"。
两者不符 = 时间戳对但画面错，那才是"前后帧跳跃"。

⚠️ **`rvfc` 会漏采**，单看帧号单调性判据不足。实测：旧版驱动
（`9f00b1a6`，无收帧线程）与修复版跑出的帧号序列逐值相同，都单调递增 ——
说明只看单调性测不出问题，必须做像素比对。

⚠️ **fMP4 有时间轴偏移**。`-movflags frag_keyframe+empty_moov` 生成的分段流
`start_time=1.0`，而原始流是 `0.0`。@2fps 就是 2 帧偏移，逐帧恒定。
比对前必须扣掉，否则 20/20 全部"不符"（实测踩过）。

## 用法

```bash
# 起回传服务器（收满指定份数即退）
OUT=/tmp/r.json ROOT=/tmp/cr WANT_REPORTS=1 python3 report.py 8871 &

# 硬解跑 MSE
LIBVA_DRIVER_NAME=msm_drm LIBVA_DRIVERS_PATH=<build目录> \
google-chrome --ozone-platform=wayland \
  --enable-features=VaapiVideoDecodeLinux,VaapiVideoDecoder,VaapiIgnoreDriverChecks \
  --autoplay-policy=no-user-gesture-required \
  'http://127.0.0.1:8871/mse_test.html?src=h264_frag.mp4&mime=video/mp4;%20codecs%3D%22avc1.64001f%22&fps=2&n=20&seek=1'
```

参数：`src` 分段流、`mime` MSE codec 串、`fps` 帧率、`n` 采样数、
`chunk` 分块字节、`seek=1` 中途插一次 seek。

## 测试页

| 页面 | 覆盖路径 | 关键参数 |
|------|----------|----------|
| `mse_test.html` | MSE `appendBuffer` 分段喂流 | `src` `mime` `fps` `n` `chunk` `seek=1` |
| `net_test.html` | 真实 HTTP 在线源直读 | `url` `fps` `n` `seek=1` |
| `abr_test.html` | ABR 分辨率切换（换 src） | 页内 `srcs` 数组、`PER` 每档帧数 |
| `session_churn.html` | 小间隔反复建/拆会话（强度压测） | `n` `gap` `only` `list` |
| `mse_midgop.html` | GOP 中段进流 + seek + 带内 `changeType()` | `a` `b` `ca` `cb` `n` `gap` `seek=0` `swap=0` |
| `multistream.html` | 多路并发 + 每轮随机 seek | `k` `n` `gap` `seek=0` `list` |
| `codec_probe.html` | 浏览器 codec 能力探测（短式/长式分开测） | 无 |

后四个是**稳定性/能力**页，不比对像素，判据在各自文件头的注释里。
前三个靠 `report.py` 回传，后三个把计数写进 `document.title`，
用 `--remote-debugging-port` 起浏览器后读 `/json/list` 的 `title` 即可，例：

curl -s http://127.0.0.1:9335/json/list

`session_churn.html?n=1400&gap=400` 就是抓出 v0.4.7 那个
`close(0)` ×24 真凶的强度（三版本 A/B 数据见该文件头注释与 `CHANGELOG.md`）。

## 素材

10 份码流就随仓库放在本目录（共 14M；`.gitignore` 里给 `*.mp4` 开了这一条例外）。
页面用的是相对文件名，所以在**本目录**起服务就能直接跑：

python3 -m http.server 8777

| 文件 | 是什么 | 谁在用 |
|---|---|---|
| `av1_854x480.mp4` | AV1 480p，libaom `-cpu-used 8 -b:v 600k -g 12` | `session_churn.html` `multistream.html` |
| `av1_1280x720.mp4` | AV1 720p，同上 | 同上 |
| `av1_1920x1080.mp4` | AV1 1080p，同上 | 同上 |
| `av1_3840x2160.mp4` | AV1 4K，SVT-AV1 preset 12 / 8M / `seq_level_idx` 12 | 4K 强度验证（v0.4.7 已验证项） |
| `av1_854x480_frag.mp4` | 480p 的**分段**版（`frag_keyframe+empty_moov+default_base_moof`） | `mse_midgop.html` 的 `a` |
| `av1_1280x720_frag.mp4` | 720p 的分段版 | `mse_midgop.html` 的 `b`（带内 `changeType()` 换分辨率的目标） |
| `av1_hi10_1280x720.mp4` | AV1 **10-bit**，libaom | 10-bit 已知限制的复现 |
| `av1_hi10_svt_1280x720.mp4` | AV1 10-bit，SVT preset 10（`seq_profile=0 / high_bitdepth=1 / mono_chrome=0 / level 5`） | 同上，换编码器对照 |
| `av1_probe.mp4` | 像素回归用的探针流 | `tests/regress_av1_pixels.sh` 一类逐帧比对 |
| `h264_probe.mp4` | H.264 对照探针 | 同上 |

10-bit 那两份是**当前已知限制的复现材料**：解码本身正确，但浏览器拿不到画面
（`vaExportSurfaceHandle` 缺 10-bit 的 `SEPARATE_LAYERS` 导出路径，而 Chrome 实测传的就是
`READ_ONLY \| SEPARATE_LAYERS`）。做 P010 分层导出那一步时要用它们验收。

不想用仓库里这份就自己生成，各压测页文件头注释里有对应的 ffmpeg 命令。
注意两件事：**MSE 那两个页必须是分段 mp4**，普通 mp4 喂不进 `appendBuffer`；
`ca`/`cb` 里的 `seq_level_idx` 要和码流实际等级一致，
用 `ffmpeg -i x.mp4 -c:v copy -bsf:v trace_headers -f null - 2>&1 \| grep -m1 seq_level_idx` 查。

### 不依赖驱动日志判"硬解到底在不在跑"


图标启动的 Chrome 通常没有 `DMD_VA_LOG`，这时看谁占着解码器节点：

for p in $(pgrep -f "/opt/google/chrome/chrome"); do ls -l /proc/$p/fd 2>/dev/null | grep -oE "/dev/video[0-9]+"; done

有 `/dev/video32`（Venus 无状态解码节点，只有 VA-API 驱动会开）= 硬解在跑；
空了 = 已回落软解。这条在排查"站点选了 AV1 却播 H.264"时先把驱动侧排除掉。


### 实测结果（修复版驱动）

真实在线源 `jinjie_265.mp4`（HEVC 1280x720，用户最初报问题的 URL）：
- 直读播放 24 帧：单调递增、零回退，231 次配对
- 加 seek 干扰 24 帧：帧号 0-11 连续，seek 后 86-97 连续，零回退零丢帧

ABR 分辨率切换（640x360 → 1280x720 → 640x360）：
- 三档各采 10 帧，分辨率正确跟随，全部单调、零回退
- 驱动建 3 个会话（`CAPTURE 就绪` 640x368 两次、1280x720 一次），86 次配对

## 编写这类页面的两个坑

**换 src 会取消 pending 的 `requestVideoFrameCallback`。** 紧跟 `v.src=`
立刻重新注册也无效 —— 那时新流还没就绪。必须等 `loadeddata` 事件。
实测症状是整个测试卡住不回传（只建了 1 个会话）。

**MSE 播放推进依赖 `updateend` 驱动 `play()`。** 整段很快 `endOfStream`
之后 `updateend` 不再触发，若那时 `readyState` 还没到 `HAVE_CURRENT_DATA`
就没人调 `play()` 了。实测只采到 1 帧就 timeout。补 `loadeddata`/`canplay`
监听与定时兜底。

## 已知环境限制

- **Chrome MSE 拒收 HEVC**：同一个 `hevc_frag.mp4` 走 `<video src>` 直读
  能正常硬解（实测 131 次配对、帧号单调），走 MSE 报 `sb error` +
  `InvalidStateError`。这是 Chrome MSE 层的策略，与驱动无关。
  HEVC 的浏览器验证请用直读方式。
- **Chrome 软解不支持 HEVC**（专利），所以 HEVC 做不了软/硬解 A-B 对照。
  需要 A-B 对照时用 H.264 素材。
