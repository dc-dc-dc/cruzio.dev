---
title: "Translating youtube into text in seconds with whisper.cpp"
description: Embark on an exploration of Go's simplicity and speed, coupled with Whisper.cpp's transformative capabilities.
keywords: machine learning, whisper.cpp, metal, youtube
date: 2023-08-09
params:
    ShowShareButtons: true
---

## Introduction
In today's digital age, multimedia content is ubiquitous, and extracting valuable insights from such content has become increasingly important. One way to achieve this is by converting speech in videos into text, enabling easy analysis and accessibility. In this blog post, I'll walk you through the process of creating a Go application that leverages the power of Whisper.cpp, a machine learning model for speech-to-text conversion, to extract text from YouTube videos. Let's dive into the fascinating world of multimedia and programming!

You can check out the repo [here](https://github.com/dc-dc-dc/scooby)

### Retrieve YouTube Videos
Taking advantage of a great cli tool [youtube-dl](https://github.com/ytdl-org/youtube-dl), I can easily download videos from YouTube. The following command will download the video and save it as `video.mp4` in the current directory.

```bash
youtube-dl -f bestaudio -x --audio-format wav --audio-quality 0 --output ./wavs/%(id)s.%(ext)s url
```

Wrapping this using gos `os/exec` package, I can easily call this command from my Go application.

```go
cmd := exec.CommandContext(ctx, "youtube-dl", "-f", "bestaudio", "-x", "--audio-format", "wav", "--audio-quality", "0", "--output", "./wavs/%(id)s.%(ext)s", url)
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
```

### Setup Audio for Whisper
Whisper requires that audio formats meet a couple requirments, 
1. The audio must be in wav format
2. The sample rate must be 16kHz
3. There must be only one channel

To meet these requirements, I used ffmpeg to convert the audio to the correct format. The following command will convert the audio to the correct format and save it as `audio.wav` in the current directory.

```bash
ffmpeg -i video.wav -acodec pcm_s16le -ac 1 -ar 16000 audio.wav
```

Wrapping this using gos `os/exec` package, I can easily call this command from my Go application.

```go
cmd := exec.CommandContext(ctx, "ffmpeg", "-i", "video.wav", "-acodec", "pcm_s16le", "-ac", "1", "-ar", "16000", "audio.wav")
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
```

Now that we have the audio in the correct format, we can load it into the code using a [wav library](github.com/go-audio/wav) and pass it over to whisper.cpp.

### The Magic: Whisper.cpp 

Thankfully, whisper.cpp already provides golang bindings so this step was straightforward. I just had to create the model and pass in the audio.

```go
model, err := whisper.New("./whisper.cpp/models/ggml-base.en.bin")
if err != nil {
    log.Printf("[error] loading model err: %s \n", err)
    os.Exit(1)
}
defer model.Close()
context, err := model.NewContext()
if err != nil {
    fmt.Printf("[error] creating context err: %s \n", err)
    return
}
context.ResetTimings()
if err := context.Process(data, cb, nil); err != nil {
    fmt.Printf("[error] processing err: %s \n", err)
    return
}
```

Then the only thing left to doi was save the results. To keep it simple i just appended the segments to a list and then saved it to a json file.

```go
sgs := []Segment{}
cb := func(segment whisper.Segment) {
    fmt.Printf("%02d [%6s->%6s] %s\n", segment.Num, segment.Start.Truncate(time.Millisecond), segment.End.Truncate(time.Millisecond), segment.Text)
    sgs = append(sgs, TextInfo{
        StartTime: int(segment.Start.Milliseconds()),
        EndTime:   int(segment.End.Milliseconds()),
        Text:      segment.Text,
    })
}
...
f, err := os.Create(fmt.Sprintf("./wavs/%s.json", id))
if err != nil {
    fmt.Printf("[error] creating json file err: %s \n", err)
    return
}
defer f.Close()
if err := json.NewEncoder(f).Encode(sgs); err != nil {
    fmt.Printf("[error] encoding json file err: %s \n", err)
    return
}
```

### Challenges and Optimizations:
The longest part of this whole process by far was the video download, a simple optimization like streaming or saving in chunks and processing could speed this process up. Also looking at compiling whisper.cpp with the accelerator of your choice can provide some help although it already outperformed my expectations. 