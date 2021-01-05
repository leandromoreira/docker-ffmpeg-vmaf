# VMAF

[VMAF](https://github.com/Netflix/vmaf) is a perceptual video quality assessment algorithm developed by **Netflix**. It seeks to reflect the viewer’s perception of the streaming quality.

It's a valuable quality metric that can be used to computerize the common media workflow quality analysis instead of relying on people's eyes.

Its common usages, best practices, tips are explained here. Most of the information here is based on the [main repository](https://github.com/Netflix/vmaf) and the [latest post from Netflix](https://netflixtechblog.com/vmaf-the-journey-continues-44b51ee9ed12).

# TLDR; how to use

```bash
# all commands were performed using
# MacOS 10.15.4 - 2.3 GHz Quad-Core Intel Core i5
# Docker Desktop 2.3.0 / 4 logical cores / 2 GB RAM

# PLEASE PROVIDE two video files for comparison
#  ref.mp4 - the reference video file
#  distor.mp4 - the distorted videoo file (transcoded)
# both files must be at the root where you ran these commands

# analyzing all frames (from 1m 1080p video)
time docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/ref.mp4 -i /files/distor.mp4 \
  -lavfi libvmaf=log_fmt=json -f null -

Exec FPS: 14.318552
VMAF score = 93.130018
0.08s user 0.16s system 0% cpu 4:13.55 total

# (QUICKER) analyzing 1 out of 5 frames (from 1m 1080p video)
time docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/ref.mp4 -i /files/distor.mp4 \
  -lavfi libvmaf=log_fmt=json:n_subsample=5 -f null -

Exec FPS: 58.999069
VMAF score = 93.130807
0.05s user 0.12s system 0% cpu 1:02.25 total
# we make it faster (by using subsample) but the VMAF score is almost the same


# analyzing different renditions
docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/ref.mp4 -i /files/distor_lower_resolution.mp4 \
  -filter_complex \
  "[1:v]scale=1920x1080:flags=bicubic[main];[main][0:v]libvmaf=log_fmt=json:n_subsample=5" -f null -


```


## Common workflow

VMAF compares a reference video file against a targe one, usually, the following steps are taken:

1. **fetch** the reference video file
1. **apply transformations** to target video file
	* mostly: 
		* transcode (changing codec),
		* transize (changing resolution), 
		* and transrating (changing bitrate).
1. run the vmaf **comparison**
1. check the **score** (>= 70 good/fair)
1. if it's not good, go back to step 2, change the transformation's properties

![workflow](/images/common_workflow.png)

## Toolset

Although VMAF can be used as standalone software, it's easier and faster to use it as a dockerized FFmpeg filter. 

* It's faster because it takes advantage of the FFmpeg's intermediate steps (decoding/scaling) to calc VMAF, avoiding multiple passes and unnecessary storage.
* It's easier because it's a single step/tool process.

The docker image we're going to use for FFmpeg/VMAF is [five82/ffmpeg-git](https://github.com/five82/ffmpeg-git) and for FFmpeg/media transformation is [jrottenberg/ffmpeg](https://github.com/jrottenberg/ffmpeg).

## Preping enviroment

To practice the proposed workflow, we need to have a video file, for that matter let's download the big buck bunny.

```bash
# download the file we'll use throught the examples
# HUGE download ahead 330MB
wget http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_60fps_normal.mp4

# using "standard" naming
# <name/id>_<duration in s>_<fps>_<height>_<bitrate in k>.<format>
mv bbb_sunflower_1080p_60fps_normal.mp4 bunny_634s_60fps_1080p_4487kb.mp4
```

Working with a 10m video long, for learning purpose, might be slow/boring, let's generate an 1m short clip.

```bash
# generating a random 1 minute short clip
# not changing codec though
docker run --rm -v $(pwd):/files jrottenberg/ffmpeg  \
  -i /files/bunny_634s_60fps_1080p_4487kb.mp4 \
  -ss 00:01:24 -t 00:01:00 -c copy \
  /files/bunny_60s_60fps_1080p_4487kb.mp4

```

## Getting the reference

The reference file `bunny_60s_60fps_1080p_4487kb.mp4` has the following properties.

| Property   |      Value      |
|----------|:-------------:|
| codec |  h264 |
| bitrate |    4487 Kb   |
| resolution | 1080p |
| container |  mp4 |
| duration |  10s |


## Applying transformations

Toward to deliver video for all devices in diverse network conditions, the media needs to pass through some transformations, mostly related to the number of bits required and the resolution.

#### Transrating

```bash
# same resolution but different bit rate
docker run --rm -v $(pwd):/files jrottenberg/ffmpeg  \
  -i /files/bunny_60s_60fps_1080p_4487kb.mp4 \
  -c:v libx264 -crf 23 -maxrate 2M -bufsize 4M \
  /files/bunny_60s_60fps_1080p_2000kb.mp4
```
| Property   |      Value      |
|----------|:-------------:|
| codec |  h264 |
| bitrate |    2000 Kb   |
| resolution | 1080p |
| container |  mp4 |
| duration |  10s |


#### Transizing

```bash
# changing resolution
docker run --rm -v $(pwd):/files jrottenberg/ffmpeg \
  -i /files/bunny_60s_60fps_1080p_4487kb.mp4 \
  -vf scale=640:-1 -sws_flags lanczos \
  -c:v libx264 -crf 23 -maxrate 600k -bufsize 1200k \
  /files/bunny_60s_60fps_640p_600kb.mp4
```

| Property   |      Value      |
|----------|:-------------:|
| codec |  h264 |
| bitrate |    600 Kb   |
| resolution | 640p |
| container |  mp4 |
| duration |  10s |


## Run VMAF

### All frames - slower
```bash
time docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/bunny_60s_60fps_1080p_4487kb.mp4 \
   -i /files/bunny_60s_60fps_1080p_2000kb.mp4 \
   -lavfi libvmaf=log_fmt=json -f null -

Exec FPS: 14.318552
VMAF score = 93.130018
0.08s user 0.16s system 0% cpu 4:13.55 total
```
### Subsampling frames - faster (1 out of 5)
```bash
time docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/bunny_60s_60fps_1080p_4487kb.mp4 \
   -i /files/bunny_60s_60fps_1080p_2000kb.mp4 \
  -lavfi libvmaf=log_fmt=json:n_subsample=5 -f null -

Exec FPS: 58.999069
VMAF score = 93.130807
0.05s user 0.12s system 0% cpu 1:02.25 total
```

Faster (almost 4x) and still kept the close VMAF score.

### Different resolution

```bash
time docker run --rm -v $(pwd):/files five82/ffmpeg-git \
  ffmpeg -i /files/bunny_60s_60fps_1080p_4487kb.mp4 \
  -i /files/bunny_60s_60fps_640p_600kb.mp4 \
  -filter_complex \
  "[1:v]scale=1920x1080:flags=bicubic[main];[main][0:v]libvmaf=log_fmt=json:n_subsample=5" -f null -

```

### How about different container check?
You need to conform the timing information.

```bash
docker run --rm -v $(pwd):/files jrottenberg/ffmpeg -w /files \
  -i main.mpg -i ref.mkv \
  -lavfi "[0:v]settb=AVTB,setpts=PTS-STARTPTS[main];[1:v]settb=AVTB,setpts=PTS-STARTPTS[ref];[main][ref]libvmaf=psnr=1:log_fmt=json" -f null -
```
Example taken from [FFmpeg's site](https://ffmpeg.org/ffmpeg-all.html#Examples-115).
### Does it work for mobile, 4k view port?

For [4k, you can check FAQ](https://github.com/Netflix/vmaf/blob/master/FAQ.md#q-will-vmaf-work-on-4k-videos) and for mobile you can [check the available models](https://github.com/Netflix/vmaf/blob/master/resource/doc/models.md#predict-quality-on-a-cellular-phone-screen).
