---
layout: post
title: "I Created SubtitleYC"
date: 2026-08-05 21:00:00 +1000
description: I built a Windows app that turns burned-in video captions into editable subtitle files.
categories: projects ocr subtitles windows
---

**Table of contents**

- [I Created SubtitleYC](#i-created-subtitleyc)
- [Why make a new interface for OCR?](#why-make-a-new-interface-for-ocr)
- [The Overall Design](#the-overall-design)
- [Getting a video into the app](#getting-a-video-into-the-app)
- [Selecting what matters](#selecting-what-matters)
- [Turning OCR results into subtitle cues](#turning-ocr-results-into-subtitle-cues)
- [The Subtitle Editor](#the-subtitle-editor)
- [Packaging](#packaging)
- [What I learned](#what-i-learned)

## I Created SubtitleYC
Some videos have captions permanently drawn or burned into the video and sometimes, people want to transcribe those captions to translate them or extract the hardsubs to share with others.

But for videos with hardsubs, there usually isn't a proper subtitle track to extract. You could manually transcribe the subtitles and time each line to match exactly when it appears in the video. However, that process simply isn't sustainable, so we need a more efficient solution.

The obvious answer is OCR, but applying OCR to a video is not quite the same as applying it to a screenshot. A useful subtitle file needs the right text *and* the right timing. Subtitling is somewhat niche despite being used everywhere, and tools that combine hardsub OCR, frame review, editing, and export in one workflow aren't exactly a dime a dozen.

<div style="border: 1px solid #000; background-color: #aad3ff; border-radius: 4px; padding: 1rem; margin: 1.5rem 0;" markdown="1">
With that in mind, my struggle with hardsubs led me to create [SubtitleYC](https://github.com/BambooYC123/SubtitleYC), a local Windows app that turns burned-in captions into editable `.srt`, `.txt`, or `.ass` files for everyone to use.
</div>

This is the kind of large and meaningful project that I wanted to include in my blog.

## Why make a new interface for OCR?

[VideOCR](https://github.com/timminator/VideOCR) already does the difficult recognition work. It can inspect frames, run PaddleOCR over the subtitle region, compare frames, and produce timing cues. It is extremely useful, but I wanted something that suited my own workflow of creating translated English subtitles for my YouTube channel, which has 3.5k subscribers (the channel's a secret!).

A command-line OCR run is only one part of the actual workflow. Before that, I need to obtain the video, inspect it, find the caption region, and choose the right recognition settings. Afterwards, I need to watch the video again, find OCR mistakes, adjust timings, and then export the result. Doing each small step in a separate program felt like a pipeline I had to reconstruct every time. As someone who once specialised in CI/CD pipelines, I needed a solution.

**So the main idea behind SubtitleYC was to put the whole workflow in one place:**

1. Open or download a video with yt-dlp.
2. Draw a crop around the burned-in captions.
3. Run VideOCR.
4. Obtain a frame-accurate match of the subtitles alongside a video preview.
5. Use my Subtitle Editor to correct the text and timing cues.
6. Export the subtitles.

I should also add a disclaimer about AI usage: although it can help produce a surprising amount of code, I reckon a desktop app that downloads videos, starts native processes, handles large files, and edits timed data can still be too complicated for vibe coding alone. That's why I was able to use my software engineering experience, understanding of architecture, and a little AI assistance to build SubtitleYC while maintaining a clear understanding of how all those pieces were supposed to fit together.

![SubtitleYC main workspace showing recent projects](https://raw.githubusercontent.com/BambooYC123/SubtitleYC/main/docs/screenshots/01-project-library.png)

*The main workspace keeps video input, OCR settings, preview, and recent projects together.*

## The Overall Design

SubtitleYC is both a desktop shell and a small web application.

- The backend is written in Python with FastAPI. It manages the projects, launches `yt-dlp`, calls FFmpeg and VideOCR, parses subtitle files, and exposes the state needed by the interface.
- The frontend is just HTML, CSS, and JavaScript.
- PySide6 provides the Windows window and embeds the interface.
- The PyAV video surface handles accurate frame stepping, which I improved to create smooth video scrubbing.

The two halves communicate through a private server bound to `127.0.0.1`. A random port and a private token are created for each desktop session. This sounds slightly roundabout for a local application, but the split has been useful: the backend is good at process and file management, while the frontend is good at building a fairly dense editing interface.

**At a high level, the data flow is:**

```text
Video URL or local file
          |
          v
  yt-dlp / local copy
          |
          v
  FFprobe metadata and preview
          |
          v
  Crop coordinates + OCR settings
          |
          v
     VideOCR CLI
          |
          v
  Timed subtitle cues
          |
          v
 Review, edit, and export
```

## Getting a video into the app

There are two input paths. A local video can be selected directly, while a URL is passed to `yt-dlp`. For URLs, SubtitleYC first checks the available formats and can either choose automatically or let me select a particular video and audio combination. It also supports importing a site's own subtitle tracks when they are available—this is a subtitling app, so it wouldn't be right to leave out the ability to download a video's existing subtitle transcripts.

After the file is available, `ffprobe` reads its duration, dimensions, frame rate, and stream information. This metadata matters later. A timestamp that looks correct to the nearest millisecond may still fall between actual frames, so the editor can snap subtitle boundaries back to the video's frame grid.

Downloaded and uploaded files live inside a per-user workspace rather than the application directory. I found this to be the most comfortable design for storing files.

Packaged builds use:

```text
%LOCALAPPDATA%\SubtitleYC\workspace
```

Projects, preview frames, generated subtitles, settings, and logs are separated into their own folders. Keeping changeable data outside `Program Files` also means that installing a new build does not destroy an existing project.

## Selecting what matters

Running OCR over an entire video screen is slower and less accurate. A frame may contain signs, user-interface labels, watermarks, or any other text that is not dialogue. SubtitleYC therefore lets me scrub through the video and draw a rectangle around the caption area to help it read only the subtitles. The rectangle is stored as crop coordinates and reused for the OCR job.

This is one of the simpler parts of the interface that also makes the largest practical difference. The recognition engine receives fewer irrelevant pixels, and I do not need to calculate FFmpeg crop values manually.

![SubtitleYC extracting subtitles from a selected video region](https://raw.githubusercontent.com/BambooYC123/SubtitleYC/main/docs/screenshots/02-ocr-workflow.png)

*The crop box and OCR settings are passed to the real VideOCR command-line runtime. The activity panel reports progress while it runs.*

SubtitleYC has VideOCR settings that can be adjusted based on preference. These include the OCR language, confidence threshold, text similarity, frames to skip, minimum cue duration, merge gap, brightness threshold, and timing offset. The defaults are what I find useful, but the right values depend heavily on the source video.

For example, sampling fewer frames can make a run much faster, but a line may be missed. A generous similarity threshold may correctly join slightly different OCR readings of the same caption, but it can also merge lines that should remain separate. The app makes these trade-offs adjustable; there is no universally correct value because each source videos and user preferences differ.

## Turning OCR results into subtitle cues

VideOCR performs the recognition and produces timed output. SubtitleYC then normalises that output into cues with three pieces of information:

```text
start time + end time + text
```

That common representation is also what makes several output formats possible. SubRip (`.srt`) stores numbered timed cues. Advanced SubStation Alpha (`.ass`) supports more richly styled subtitle scripts. Plain text (`.txt`) contains only the recognised dialogue.

SubtitleYC can also import existing `.srt`, `.ass`, or `.ssa` files into the current video session. Plain text is export-only because there is no timing information.

OCR is never perfectly reliable, especially with stylised fonts, outlines, motion, low resolution, or captions placed over a changing background. For that reason, I think the editor is at least as important as the OCR button. Based on some feedback, I will attempt to improve the OCR's handling of stylised fonts. I will also add subtitle styling options in the subtitle editor in future releases.

## The Subtitle Editor

The editor shows the video and cue list together. Selecting a cue seeks to its position, and the video controls can move one frame at a time. A cue's text, start, and end can be edited directly; cues can also be inserted or deleted.

![SubtitleYC Editor reviewing subtitle cues against the video](https://raw.githubusercontent.com/BambooYC123/SubtitleYC/main/docs/screenshots/03-subtitle-editor.png)

*The editor is where OCR output becomes a subtitle file I would actually use.*

The timing controls ended up being more involved than I first expected.

It is useful to be able to:

- nudge only the start or end of a cue by a chosen number of frames;
- move every cue earlier or later when the whole file has a fixed offset;
- snap cue boundaries to real video frames;
- jump to the previous or next subtitle boundary; and
- undo or redo edits without repeatedly saving backup files.

There are two preview implementations. In the packaged desktop app, PySide6 and PyAV provide a native surface with an in-memory frame cache. A web canvas remains as a fallback. The native path is more work, but it makes frame-by-frame review feel smoother and more like part of a desktop editor than a video element inside a browser.

## Packaging

The Python code is only a small part of what a downloadable OCR app needs. A useful Windows build also needs FFmpeg, FFprobe, VideOCR, PaddleOCR models, Qt, and their corresponding licences. GPU acceleration complicates this further because different NVIDIA generations may need different CUDA builds.

That is why, instead of putting every runtime into one enormous installer, I created separate editions of SubtitleYC. The CPU edition is the default and works without an NVIDIA GPU. Each CUDA edition includes a matching GPU runtime. Installing another edition upgrades the same application and replaces the previous OCR runtime instead of leaving several multi-gigabyte copies behind.

I recommend a GPU edition if you have a supported NVIDIA system, since GPU-accelerated OCR is much faster than the CPU version, as you would expect.

Release builds also collect third-party notices, record hashes for bundled tools, run dependency checks, and can require Windows code signing.

None of this changes the OCR algorithm, but it does make life easier for anyone wanting to install SubtitleYC.

## What I learned

The main lesson was that even combining existing tools is difficult. VideOCR already solved the central recognition problem. Most of the work in SubtitleYC came from the less interesting parts: handling failed downloads, keeping the interface responsive during long jobs, preserving projects between upgrades, seeking to exact frames, validating file paths, stopping subprocesses, and making the generated output easy to correct. I could go on and on, but I'll keep my ranting out of the blog.

I also ended up with a better appreciation for local software tools and their creators. Keeping the video and OCR workflow on the user's PC avoids uploading large or private media to another service, but local applications have their own security boundary. The loopback API still needs authentication and origin checks, downloaded files still need to be treated as untrusted, and bundled dependencies still need updates.

SubtitleYC is still in early access, so generated subtitles should be checked and important projects should be backed up.

It is open source and available under the MIT License (yay!) on [GitHub](https://github.com/BambooYC123/SubtitleYC).

There are still rough edges that need to be addressed, but for now it covers the complete workflow: downloading or uploading a video, extracting its burned-in captions, editing them for accuracy or flair, and exporting them for personal use. A task that previously took multiple hours now takes about five minutes and requires only a few clicks.

**Reducing that workload for myself—and potentially for other people—is exactly why I made the app in the first place.**
