Hello, I've made the movie watching world inside VRChat. Here's the link and overview.

https://vrchat.com/home/world/wrld_583ae9b6-e8f5-4806-a560-2b19782a1f83

It has audiolink and capable of 7.1 and 5.1 surround sound, Dolby Atmos 5.1.2, and normal Youtube video, VRCDN also support.

# Prepairing your movie files for the world

*Some rant  
[VideoTXL](https://github.com/vrctxl/VideoTXL) and any other video players inside VRChat use AVpro, which rely on Windows Media Foundation. If you were asking what format it supports from the [official documentaion](https://learn.microsoft.com/en-us/windows/win32/medfound/supported-media-formats-in-media-foundation), You'll find that it's kinda disappointing as windows didn't update their media component since Windows 7, and rely on [Media extention plugin](https://apps.microsoft.com/store/detail/hevc-video-extensions/9NMZLZ57R3T7) for newer codecs.*

*Very first-first,* Some knowledge about command-line, ffmpeg and AV format is require, if you don't know about this. Pass this to your nerd friends to do it for you.  

First, you must obtain a movie file. (rather rip your own Blu-ray or DVD, or download some if it online.)  
Second, install [ffmpeg](https://www.wikihow.com/Install-FFmpeg-on-Windows)  
Third, A server that can be able to serve files.  
In this tutorial I'll use `input.mkv` as source movie file and `Movie.mp4` as a result file for playing inside VRChat.  

## Video
You might need to re-encode your video to be able to watch it with your friends inside vrc. If your video is already H264 encoded with 1080p, yuv420p pixel format, and has resonable bitrate (4-10 mbps). You can just skip the encodeing part and do a stream copy.  


###Stream copy

```
ffmpeg 
  -i "input.mkv" \
  -map 0:v \
  -map 0:a \
  -c copy \
  output.mp4
```


### x264 software encoder
```
ffmpeg -hwaccel cuda \
  -i "input.mkv" \
  -vf scale=1920:-2:flags=lanczos \
  -map 0:v \
  -c:v libx264 \
  -preset slow \
  -profile:v high -pix_fmt yuv420p \
  -g 95.88 \
  -crf 23 -maxrate 6M -bufsize 12M \
  output.mp4
```

###Explain this (for nerd)
  
  `-hwaccel cuda` use CUDA for decoding a video, remove this if you don't have Nvidia GPU  
  `-vf scale=1920:-2:flags=lanczos` Scale a video to 1080p, max recommended for vrc video.  
  `-map 0:v` We'll only encode the video part of the file (for now)  
  `-c:v libx264` Use X264 for encoding  
  `-preset slow ` Use slow preset, it can be `ultrafast,superfast,veryfast,faster,fast,medium,slow,slower,veryslow,placebo` Set this to balance between speed and time, from fastest to slowest with worst to best compresstion.  
  `-profile:v high -pix_fmt yuv420p` Use high prefole and yuv420p for pixel format. **Pixel format other than yuv420p will crash vrc.**  
  `-g 95.88` 4 second GOP, you might wanted to adjust this to fir your movie fps. `FPS*4 for GOP size`   
  `-crf 23 -maxrate 6M -bufsize 12M` Change your prefered bitrate here, for 1080p video I recommned max bitrate at 6Mbps for 1080p video. `bufsize` should be twice the max bitrate. and crf 23 should be good enough.


### NVENC Hardware encoder (Nvidia)
```
ffmpeg -hwaccel cuda \
  -i input.mkv \
  -vf scale=1920:-2:flags=lanczos \
  -map 0:v \
  -c:v h264_nvenc \
  -preset p7 -profile:v high -rc vbr_hq -pix_fmt yuv420p \
  -bf 2 -g 95.88 -rc-lookahead 48 \
  -2pass 1 -multipass fullres -no-scenecut 1 -spatial-aq 1 -temporal-aq 1 \
  -cq 27 -maxrate 6M -bufsize 12M \
  video.mp4
```
###Explain this (for nerd)
  
  `-hwaccel cuda` use CUDA for decoding a video  
  `-vf scale=1920:-2:flags=lanczos` Scale a video to 1080p, max recommended for vrc video.  
  `-map 0:v` We'll only encode the video part of the file (for now)  
  `-c:v h264_nvenc` Use NVENC H264 for encoding  
  `-preset p7 -profile:v high -rc vbr_hq -pix_fmt yuv420p` Use p7 preset, high profile, vbr rate control, and yuv420p for pixel format. **Pixel format other than yuv420p will crash vrc.**  
  `-bf 2 -g 95.88 -rc-lookahead 48` insert 2 B-frame, 4 second GOP, Lookahead for vbr rate control for 48 frames. you might wanted to adjust this to fir your movie fps. `FPS*4 for GOP size` `FPS*2 for rate control`  
  `-2pass 1 -multipass fullres -no-scenecut 1 -spatial-aq 1 -temporal-aq 1` NVENC thing  
  `-cq 27 -maxrate 6M -bufsize 12M` Change your prefered bitrate here, for 1080p video I recommned max bitrate at 6Mbps for 1080p video. `bufsize` should be twice the max bitrate. and 27 qp should be good enough.

x264 should produce video with better quality and file size, but nothing stopping you from putting more bit rate into your video, as it come down to how many bandwidth for your server has?  

### 4K HDR content
VRChat doesn't support HDR. There's a way to tone mapping HDR to SDR using ffmpeg.

```
ffmpeg -vsync 0 -hwaccel cuda \
  -init_hw_device opencl=ocl -filter_hw_device ocl -extra_hw_frames 3 \
  -i input.mkv \
  -map 0:v \
  -vf "format=p010,hwupload,tonemap_opencl=tonemap=mobius:param=0.01:desat=0:r=tv:p=bt709:t=bt709:m=bt709:format=nv12,hwdownload,format=nv12" \
  -s 1920x1080 \
  -c:v h264_nvenc \
  -preset p7 -profile:v high -rc vbr_hq -pix_fmt yuv420p \
  -bf 2 -g 95.88 -rc-lookahead 48 \
  -2pass 1 -multipass fullres -no-scenecut 1 -spatial-aq 1 -temporal-aq 1 \
  -cq 27 -maxrate 6M -bufsize 12M \
  video.mp4
```
You should change the `-resize 1920x1080` to the video resolution you wanted. If you set this wrong, it'll affect the aspect ratio of your movie. VRChat can play 4K video, but it's not recommended.  
 
**Keep in mind that properly mastered SDR content directly from Blu-ray will looks better than tone mapped HDR from ffmpeg.**

# Audio
Before we can do that. let's see what inside the file first, by using `ffprobe`

```
ffprobe -hide_banner -i input.mkv
```
This will be a result, and it might be different for your file.

```
Input #0, mpegts, from 'E:\Downloads\dolby-audiosphere-lossless.m2ts':
  Duration: 00:01:01.09, start: 4200.000000, bitrate: 19563 kb/s
  Program 1
  Stream #0:0[0x1011](und): Video: h264 (High) (HDMV / 0x564D4448), yuv420p(progressive), 1920x1080, 24 fps, 24 tbr, 90k tbn
  Stream #0:1[0x1100]: Audio: truehd (Dolby TrueHD + Dolby Atmos) (AC-3 / 0x332D4341), 48000 Hz, 7.1, s32 (24 bit)
  Stream #0:2[0x1100]: Audio: ac3 (AC-3 / 0x332D4341), 48000 Hz, 5.1(side), fltp, 640 kb/s
  Stream #0:3[0x1101]: Audio: eac3 (Dolby Digital Plus + Dolby Atmos) (AC-3 / 0x332D4341), 48000 Hz, 7.1, fltp, 1280 kb/s
```

## My file has 5.1 Surround sound audio
If your file is EAC-3 or AC-3 encoded, I have good news for you. You can play E-AC-3 inside vrchat as Windows 10 MF (Media Foundation) supported that. *(This is undocumented by Microsoft. It could be that I'm using Windows 10 Pro, or I installed [k-lite codec pack](https://codecguide.com/download_kl.htm) or other factors)*  
To do this you simply remux `video.mp4` with your audio track from `input.mkv`  

In this exsample from ffprobe, Look for `5.1` and `ac3` or `eac3`, that codec should be playable inside vrchat. In my case is `Stream #0:2` So you'll need to put `-map 1:a:1` under this command (Index starts at 0)  

```
ffmpeg \
  -i movie.mp4 \
  -i input.mkv \
  -map 0:v \
  -map 1:a:2 \
  -c copy \
  -movflags +faststart \
  Movie.mp4
```
You're done and ready, put this on your server and URL inside video player.  
### My file is DTS or Truehd encoded audio
DTS audio doesn't support inside VRChat. You can convert that into `aac` by using this ffmpeg command.
```
ffmpeg \
  -i movie.mp4 \
  -i input.mkv \
  -map 0:v \
  -map 1:a:2 \
  -c:v copy \
  -c:a aac \
  -b:a 448k \
  -movflags +faststart \
  Movie.mp4
```
Reminder that this only work for 5.1 audio, If you have 7.1 follow the next section

## My file has 7.1 Surround sound audio
If your file is EAC3 encoded, you can do the same procedures as above. If your file is DTS or Truehd encoded audio, thing started to get difficult, becuase Media Foundation doesn't support 7.1 encoded AAC file.  
 > This is due to [fraunhofer's implementation of AAC multichannel](https://www2.iis.fraunhofer.de/AAC/multichannel.html) 7.1 Channel Support are added in 2013 as AAC Amendment 4, But Windows didn't update Media Foundation component since Windows 7 (2008-9 ish). So, we stuck with 5.1 max channel support.  
The work around is to find the E-AC-3 encoder. Unfortunately ffmpeg has broken imprementaion of E-AC-3 and only support up to 5.1 channels.  

The only way to use 7.1 without E-AC-3 codec is to use `pcm` audio with `mov` container, Nothing will stop you from using that except you and your friends has limited internet bandwidth, and server cost.


###ffmpeg command

```
ffmpeg \
  -i movie.mp4 \
  -i input.mkv \
  -map 0:v \
  -map 1:a:2 \
  -c:v copy \
  -c:a pcm_s16le \
  -movflags +faststart \
  Movie.mov
```

If you really wanted to get E-AC-3 inside the file. The only method is to use propriety software (ugh) called Adobe Audition CC 2017. It's needs to be 2017 version of Adobe Audition, because that the last version to support E-AC-3 encoding.

##Adobe Audition CC 2017 with E-AC-3
First, you extract audio from your movie.

```
ffmpeg \
  -i input.mkv \
  -map 1:a:0 \
  audio.flac
```

Then open that `audio.flac` file inside Adobe Audition. Hit `Ctrl + Shit + S` to bring up save as dialog.  
Select Dolby Digital  
![Adobe_Audition_CC_2iam1pR1aA](https://user-images.githubusercontent.com/136845818/257661067-45e8dd61-c6bc-4c2c-a74c-d002779aa28d.png)


And click on format settings  
![Adobe_Audition_CC_Jb0ooz7MlM](https://user-images.githubusercontent.com/136845818/257661144-641f9add-f2c9-49c6-913b-abe59a735de9.png)

Put in the same settings follow this image  
![Adobe_Audition_CC_OFkAhK5oQz](https://user-images.githubusercontent.com/136845818/257661164-a44c8fa1-b221-45b2-ac69-34cb32db1d8d.png)


Then, go Pre-Processing and uncheck everything.  
![Adobe_Audition_CC_SoKVz182KO](https://user-images.githubusercontent.com/136845818/257661185-7c539cba-494e-429d-a449-0e782d9778c8.png)

Hit ok and save it somewhere. In this case I'll save it as `audio.ec3`  

Now move on to another ffmpeg command.  
```
ffmpeg \
  -i movie.mp4 \
  -i audio.ec3 \
  -map 0:v \
  -map 1:a \
  -c copy \
  -movflags +faststart \
  Movie.mp4
```
Done, you're now getting movie inside VRChat with fancy 7.1 E-AC-3 codec.

# Dolby Atmos
There's no decoding for Dolby Atmos inside VRChat. Unless you use [Cavern](https://cavern.sbence.hu/cavern/) to decode Dolby Atmos into 5.1.2 channel, and select 5.1.2 audio preset inside the video player.

The downside of Cavern is, It's buggy, not fully implemented all the Atmos function yet. And sometimes blasting very loud noise.
# Audio Presets
![VRChat_Dg3HK26u2Y](https://user-images.githubusercontent.com/136845818/257670502-774ee330-0499-4bc3-aa80-85a655e55a11.png)

### Surround 7.1
This is a normal 7.1 setup, reference from [7.1 Virtual speaker setup Dolby guide](https://www.dolby.com/about/support/guide/speaker-setup-guides/7.1-virtual-speakers-setup-guide/)

Same goes for 5.1, Just rear speakers unused.

### Atmos 5.1.2
This will remapped 7.1 setup to 5.1.2 reference from [5.1.2 Overhead speaker setup](https://www.dolby.com/about/support/guide/speaker-setup-guides/5.1.2-overhead-speaker-setup-guide/)

| Original channels (audio file)     | Re-mapped channels |
| ----------- | ----------- |
| Back-Left   | Top-Left    |
| Back-Right  | Top-Right   |

You might have to experiment with channel mapping from 5.1.2 in 7.1 coded audio channel. I'll write the guide soon.  

### Cavern 4.1.1
This audio preset is for using with Atmos file decoded with [Cavern](https://cavern.sbence.hu/cavern/) and encoded with 5.1 audio channels.  

| Original channels (audio file)     | Re-mapped channels |
| ----------- | ----------- |
| Side-Left   | Top-Center  |
| Side-Right  | Back-Center |

### HRTF
Head-related transfer function  
This work by placing Left and Right speaker channels directly next to your ears. It need HRTF or Binaural stereo encoded file, and require you to sit at the center of sofa. This will sounds like you're using AirPods Pro with Dolby Atmos.

### Global
This is VideoTXL default global audio, you'll hear everything everywhere like you're wearing a headphone. Only stereo audio (Front-Left, Front-Right) will be heard.
