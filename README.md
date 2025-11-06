# AV Recorder App  
*Multithreaded Audio & Video Recorder with Speech-to-Text Integration*

## Overview  
The AV Recorder App is a Python-based desktop tool that simultaneously captures **video** (via OpenCV) and **audio** (via SoundDevice), then merges them in real time using FFmpeg. It demonstrates hands-on integration of speech recognition and multithreading to improve capture performance and eliminate manual post-processing.

## Key Features  
- 🎥 **Dual-stream capture**: record webcam video and microphone audio concurrently.  
- 🗣️ **Speech-to-text automation**: convert spoken audio to text using the `speech_recognition` library.  
- ⚡ **Multithreaded design**: decouples capture, processing, and UI threads for smoother performance.  
- 🧰 **GUI interface**: built with Tkinter for intuitive start/stop controls and automatic file naming.  
- 🧾 **Real-time muxing**: uses FFmpeg to synchronise audio and video streams into a single high-quality output.  
- 🪶 **Lightweight & modular**: adaptable for use-cases such as meeting capture, screen recording, or voice-command dataset creation.

## Tech Stack  
- **Language**: Python 3  
- **Libraries/Modules**: OpenCV, SoundDevice, SoundFile, SpeechRecognition, Tkinter, threading, datetime  
- **Tooling**: FFmpeg command-line, Git  
- **Platform**: Cross-platform (tested on Windows / Linux)

## Project Structure  
