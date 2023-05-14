---
layout: page
title: "Make AI generated MV using so-vits-svc and SadTalker"
permalink: /make-ai-generated-mv-using-so-vits-svc-and-sadtalker/
---

## Introduction

With recent innovations in AI, we can now easily generate voice matching your tone, and even make a photo animation matching the voice. In this post, I will show you how to combine them together to make AI generated music video of your own. The two main components are [so-vits-svc](https://github.com/svc-develop-team/so-vits-svc) and [SadTalker](https://github.com/OpenTalker/SadTalker).

## [so-vits-svc](https://github.com/svc-develop-team/so-vits-svc) (SoftVC VITS Singing Voice Conversion)

`so-vits-svc` can train a model of your voice, providing enough voice samples. Then it can convert any voice to your voice. You can also use some TTS service to generate voice from text, then change it to your own voice.

In this post, we are going to use a fork of it [so-vits-svc-fork](https://github.com/voicepaw/so-vits-svc-fork), which has better interface and more features.

## [SadTalker](https://github.com/OpenTalker/SadTalker)

`SadTalker` can generate a photo of face animation matching the voice. The model is pre trained, so you don't need to train it yourself. Just provide a portrait photo and an audio file, it will generate a video with talking head.

## Audio

### 1. Prepare voice samples

You need to prepare enough voice samples of your own. The more the better. The format should be `wav`.

### 2. Setup training environment

Follow instructions in [so-vits-svc-fork](https://github.com/voicepaw/so-vits-svc-fork) to setup python environment and install packages.

### 3. Pre process voice samples

Training voice samples should be placed under `dataset_raw/{speaker_id}/`. But it requires each sample to be less than 10 seconds to avoid out of memory error.

So we can split them first. Place all voice samples under `dataset_raw_raw/{speaker_id}/`. Then run `svc pre-split`.

Check results under `dataset_raw/{speaker_id}/`, remove any samples that are too short (< 2s) or too long (> 10s).

Then run some other pre process steps:

```bash
$ svc pre-resample
$ svc pre-config
$ svc pre-hubert -fm crepe
```

### 4. Train model

This step will take a long time. In my case, with RTX 4070, batch size 16, about 2k voice samples, it took about 2 minutes per epoch. I spent about 1 day to train 777 epochs.

```bash
$ svc train -t
```

You can monitor the training process in TensorBoard, it's running on port 6006 by default. [http://localhost:6006/](http://localhost:6006/)

### 5. Prepare target voice

Prepare the target voice you want to replace, either download an existing one or use TTS service to generate one. If it contains background music, you can use [Ultimate Vocal Remover](https://github.com/Anjok07/ultimatevocalremovergui) to split vocals and instruments.

I used `Demucs` model to split vocals and instruments, then used `VR Architecture` to further remove mixins from vocals to get raw vocals.

### 6. Inference

Now we have the model and raw vocals, we can convert raw vocals to model voice.


```bash
$ svc infer source.wav
```

Or using GUI to tune more parameters:


```bash
$ svcg
```

### 7. Post process

Merge the converted voice with instruments to get the final audio file. This can be easily done with [Audacity](https://www.audacityteam.org/). You can also do some audio tunning here.

## Video

### 1. Prepare portrait photos

Prepare some portrait photos of your own. It depends on how many animations you want to have in the final MV.

### 2. Setup environment

Follow instructions in [SadTalker](https://github.com/OpenTalker/SadTalker) to setup python environment and install packages.

### 3. Pre process audio

The audio file should be the vocals file we got in previous steps. If the audio is too long, you can split it into several parts, otherwise you may get out of memory error if you don't have enough GPU memory. In my case, I split it into 1 minute per file.

### 4. Generate video

```bash
$ python inference.py --driven_audio <audio.wav> \
                  --source_image <video.mp4 or picture.png> \
                  --result_dir <a file to store results> \
                  --still \
                  --preprocess full \
                  --enhancer gfpgan
```

Repeat that for each audio file and portrait photo.


## Merge audio and video

Now we have the final audio and video, we can merge them together to get the final MV. It can be done by [ffmpeg](https://ffmpeg.org/) or advanced video editors.

## Conclusion

With the help of AI, we can easily generate music video of our own. Just some voice samples and portrait photos, we can sing any song without saying one word, even in a completely different language. How cool is that!
