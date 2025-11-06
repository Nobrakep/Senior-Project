#!/usr/bin/env python3
import os
import threading
import subprocess
import time
from datetime import datetime
import tkinter as tk
from tkinter import ttk, filedialog, messagebox, simpledialog
import cv2 # type: ignore
import sounddevice as sd # type: ignore
import soundfile as sf # type: ignore
import numpy as np # type: ignore
import shutil
import sys
import webbrowser

# Optional voice recognition / STT
try:
    import speech_recognition as sr # type: ignore
    SR_AVAILABLE = True
except Exception:
    SR_AVAILABLE = False

# Path to ffmpeg binary. If None, will try to find on PATH.
FFMPEG_BIN = None  # e.g. r"C:\tools\ffmpeg\bin\ffmpeg.exe" or None to auto-detect

def find_ffmpeg():
    if FFMPEG_BIN:
        return FFMPEG_BIN if os.path.exists(FFMPEG_BIN) else shutil.which("ffmpeg")
    return shutil.which("ffmpeg")

# ---------- helpers ----------
def make_timestamped_dir(parent=None):
    ts = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    base = parent or os.getcwd()
    d = os.path.join(base, ts)
    os.makedirs(d, exist_ok=True)
    return d

def open_path_in_explorer(path):
    if not path:
        return
    if os.path.isdir(path):
        target = path
    else:
        target = os.path.dirname(path) or "."
    try:
        if sys.platform.startswith("win"):
            os.startfile(os.path.abspath(target))
        elif sys.platform == "darwin":
            subprocess.Popen(["open", target])
        else:
            subprocess.Popen(["xdg-open", target])
    except Exception:
        webbrowser.open("file://" + os.path.abspath(target))

def play_file(path):
    if not path or not os.path.exists(path):
        messagebox.showwarning("Play", "File not found.")
        return
    try:
        if sys.platform.startswith("win"):
            os.startfile(path)
        elif sys.platform == "darwin":
            subprocess.Popen(["open", path])
        else:
            subprocess.Popen(["xdg-open", path])
    except Exception as e:
        messagebox.showerror("Play error", str(e))

def voice_stop_listener(stop_event, stop_phrase="stop recording"):
    if not SR_AVAILABLE:
        return
    r = sr.Recognizer()
    try:
        mic = sr.Microphone()
    except Exception:
        return
    with mic as source:
        r.adjust_for_ambient_noise(source, duration=0.5)
        while not stop_event.is_set():
            try:
                audio = r.listen(source, timeout=5, phrase_time_limit=5)
                text = r.recognize_google(audio).lower()
                if stop_phrase.lower() in text:
                    stop_event.set()
                    break
            except Exception:
                continue

def audio_recorder_stream(stop_event, out_wav_path, samplerate=None, channels=1, blocksize=1024):
    if samplerate is None:
        try:
            info = sd.query_devices(None, 'input')
            samplerate = int(info['default_samplerate'])
        except Exception:
            samplerate = 44100
    frames = []
    def callback(indata, frames_count, time, status):
        if status:
            pass
        frames.append(indata.copy())
        if stop_event.is_set():
            raise sd.CallbackStop
    try:
        with sd.InputStream(samplerate=samplerate, channels=channels, dtype='float32',
                            blocksize=blocksize, callback=callback):
            while not stop_event.is_set():
                sd.sleep(100)
    except Exception:
        stop_event.set()
    if frames:
        data = np.concatenate(frames, axis=0)
        if data.ndim == 1:
            data = data[:, None]
        try:
            sf.write(out_wav_path, data, samplerate)
        except Exception as e:
            print("Audio write error:", e)

def video_recorder_stream(stop_event, video_path, camera_index=0, fps=30, show_preview=False):
    cap = cv2.VideoCapture(camera_index)
    if not cap.isOpened():
        print("Cannot open camera")
        stop_event.set()
        return
    if fps > 0:
        cap.set(cv2.CAP_PROP_FPS, fps)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH)) or 640
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT)) or 480
    fourcc = cv2.VideoWriter_fourcc(*'mp4v')
    out = cv2.VideoWriter(video_path, fourcc, max(1, fps), (width, height))
    try:
        while not stop_event.is_set():
            ret, frame = cap.read()
            if not ret:
                break
            out.write(frame)
            if show_preview:
                cv2.imshow("Recording (press q to stop)", frame)
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    stop_event.set()
                    break
    except Exception:
        stop_event.set()
    finally:
        cap.release()
        out.release()
        if show_preview:
            cv2.destroyAllWindows()

# ---------- Tkinter UI ----------
class RecorderApp:
    def __init__(self, root):
        self.root = root
        root.title("AV Recorder")
        self.parent_dir = tk.StringVar(value=os.getcwd())
        self.base_name = tk.StringVar(value=f"recording_{datetime.now().strftime('%Y%m%d_%H%M%S')}")
        self.phrase = tk.StringVar(value="stop recording")
        self.fps = tk.IntVar(value=30)
        self.camera = tk.IntVar(value=0)
        self.show_preview = tk.BooleanVar(value=False)
        self.shortest = tk.BooleanVar(value=True)  # stop at shortest stream when muxing
        self.status = tk.StringVar(value="Idle")
        self.ffmpeg_path = tk.StringVar(value=find_ffmpeg() or "Not found")
        self.recordings = []  # list of tuples (outdir, video, audio, muxed)

        frm = ttk.Frame(root, padding=10)
        frm.grid(sticky="nsew")
        root.columnconfigure(0, weight=1)
        root.rowconfigure(0, weight=1)

        # Top controls
        top = ttk.Frame(frm)
        top.grid(column=0, row=0, sticky="we")
        ttk.Label(top, text="Save Dir:").grid(column=0, row=0, sticky="w")
        ttk.Entry(top, textvariable=self.parent_dir, width=40).grid(column=1, row=0, sticky="we")
        ttk.Button(top, text="Browse", command=self.browse).grid(column=2, row=0)
        top.columnconfigure(1, weight=1)

        ttk.Label(frm, text="Base name:").grid(column=0, row=1, sticky="w", pady=(6,0))
        ttk.Entry(frm, textvariable=self.base_name).grid(column=1, row=1, sticky="we", pady=(6,0))

        mid = ttk.Frame(frm)
        mid.grid(column=0, row=2, sticky="we", pady=(6,0))
        ttk.Label(mid, text="FPS:").grid(column=0, row=0, sticky="w")
        ttk.Entry(mid, textvariable=self.fps, width=6).grid(column=1, row=0, sticky="w")
        ttk.Checkbutton(mid, text="Preview window", variable=self.show_preview).grid(column=2, row=0, sticky="w", padx=8)
        ttk.Checkbutton(mid, text="Mux - stop at shortest stream", variable=self.shortest).grid(column=3, row=0, sticky="w", padx=8)

        ttk.Label(frm, text="Stop phrase:").grid(column=0, row=3, sticky="w", pady=(6,0))
        ttk.Entry(frm, textvariable=self.phrase).grid(column=1, row=3, sticky="we", pady=(6,0))

        btn_frame = ttk.Frame(frm)
        btn_frame.grid(column=0, row=4, columnspan=2, pady=8, sticky="w")
        self.start_btn = ttk.Button(btn_frame, text="Start", command=self.start_recording)
        self.start_btn.grid(column=0, row=0, padx=5)
        self.stop_btn = ttk.Button(btn_frame, text="Stop", command=self.stop_recording, state="disabled")
        self.stop_btn.grid(column=1, row=0, padx=5)

        # Right side: recordings list + actions
        right = ttk.Labelframe(frm, text="Recordings / Tools", padding=8)
        right.grid(column=2, row=0, rowspan=6, padx=(10,0), sticky="ns")
        self.listbox = tk.Listbox(right, width=48, height=12)
        self.listbox.grid(column=0, row=0, columnspan=3, sticky="nsew")
        self.listbox.bind("<Double-Button-1>", lambda e: self.open_selected())
        ttk.Button(right, text="Open Folder", command=self.open_selected_folder).grid(column=0, row=1, pady=(6,0))
        ttk.Button(right, text="Play Muxed", command=self.play_selected_muxed).grid(column=1, row=1, pady=(6,0))
        ttk.Button(right, text="Transcribe Selected", command=self.transcribe_selected_muxed).grid(column=2, row=1, pady=(6,0))

        ttk.Label(frm, textvariable=self.status).grid(column=0, row=7, columnspan=3, sticky="w", pady=(6,0))
        ttk.Label(frm, text="ffmpeg:").grid(column=0, row=8, sticky="w", pady=(6,0))
        ttk.Entry(frm, textvariable=self.ffmpeg_path, state="readonly").grid(column=1, row=8, sticky="we", pady=(6,0))

        self.stop_event = None
        self.threads = []
        self.latest_video = None
        self.latest_audio = None
        self.latest_muxed = None

        # create menu and keep references to menu items we need to enable/disable
        self.create_menu(root)
        self.update_sr_availability_ui()
        self._update_menu_state_idle()

    def create_menu(self, root):
        self.menubar = tk.Menu(root)

        # File menu
        filem = tk.Menu(self.menubar, tearoff=0)
        filem.add_command(label="Open Save Folder...", command=lambda: open_path_in_explorer(self.parent_dir.get()))
        filem.add_separator()
        filem.add_command(label="Exit", command=root.quit)
        self.menubar.add_cascade(label="File", menu=filem)

        # Controls menu (Record/Stop/Play)
        controlm = tk.Menu(self.menubar, tearoff=0)
        controlm.add_command(label="Record", command=self.start_recording, accelerator="Ctrl+R")
        controlm.add_command(label="Stop", command=self.stop_recording, accelerator="Ctrl+S", state="disabled")
        controlm.add_separator()
        controlm.add_command(label="Play Last Muxed", command=lambda: play_file(self.latest_muxed))
        controlm.add_command(label="Open Last Folder", command=lambda: open_path_in_explorer(os.path.dirname(self.latest_video) if self.latest_video else self.parent_dir.get()))
        self.menubar.add_cascade(label="Controls", menu=controlm)
        self.controls_menu = controlm

        # Options
        optm = tk.Menu(self.menubar, tearoff=0)
        optm.add_command(label="Set ffmpeg Path...", command=self.set_ffmpeg_path)
        optm.add_command(label="Change Save Directory...", command=self.browse)
        self.menubar.add_cascade(label="Options", menu=optm)

        # Tools
        toolm = tk.Menu(self.menubar, tearoff=0)
        toolm.add_command(label="Transcribe Last WAV", command=self.transcribe_last_wav)
        toolm.add_command(label="Transcribe Last Muxed", command=self.transcribe_last_muxed)
        toolm.add_command(label="Open Last Muxed", command=lambda: play_file(self.latest_muxed))
        self.menubar.add_cascade(label="Tools", menu=toolm)

        # Help
        helpm = tk.Menu(self.menubar, tearoff=0)
        helpm.add_command(label="About", command=self.show_about)
        self.menubar.add_cascade(label="Help", menu=helpm)

        root.config(menu=self.menubar)

        # keyboard shortcuts
        root.bind_all("<Control-r>", lambda e: self.start_recording())
        root.bind_all("<Control-R>", lambda e: self.start_recording())
        root.bind_all("<Control-s>", lambda e: self.stop_recording())
        root.bind_all("<Control-S>", lambda e: self.stop_recording())

    def _set_controls_state(self, record_enabled=True, stop_enabled=False):
        try:
            self.controls_menu.entryconfig(0, state="normal" if record_enabled else "disabled")
            self.controls_menu.entryconfig(1, state="normal" if stop_enabled else "disabled")
        except Exception:
            pass
        self.start_btn.config(state="normal" if record_enabled else "disabled")
        self.stop_btn.config(state="normal" if stop_enabled else "disabled")

    def _update_menu_state_recording(self):
        self._set_controls_state(record_enabled=False, stop_enabled=True)

    def _update_menu_state_idle(self):
        self._set_controls_state(record_enabled=True, stop_enabled=False)

    def update_sr_availability_ui(self):
        state = "normal" if SR_AVAILABLE else "disabled"
        for w in self.root.winfo_children():
            for sub in w.winfo_children():
                for sub2 in sub.winfo_children():
                    if isinstance(sub2, ttk.Button) and sub2.cget("text") == "Transcribe Selected":
                        sub2.config(state=state)
        if not SR_AVAILABLE:
            self.status.set("Idle — speech_recognition not available; transcription disabled")

    def show_about(self):
        messagebox.showinfo("About", "AV Recorder\nMenu: Record/Stop/Play/Options/Tools/Help.\nTranscription can run on WAV or muxed MP4 (requires ffmpeg).")

    def set_ffmpeg_path(self):
        p = simpledialog.askstring("ffmpeg path", "Enter full path to ffmpeg executable (leave blank to auto-detect):",
                                   initialvalue=self.ffmpeg_path.get() if self.ffmpeg_path.get() != "Not found" else "")
        if p is None:
            return
        p = p.strip()
        global FFMPEG_BIN
        if p == "":
            FFMPEG_BIN = None
            self.ffmpeg_path.set(find_ffmpeg() or "Not found")
        else:
            if os.path.exists(p):
                FFMPEG_BIN = p
                self.ffmpeg_path.set(p)
            else:
                messagebox.showwarning("ffmpeg", "Path does not exist. Keeping previous value.")
        self.root.update_idletasks()

    def browse(self):
        d = filedialog.askdirectory(initialdir=self.parent_dir.get())
        if d:
            self.parent_dir.set(d)

    def start_recording(self):
        if self.stop_event and not self.stop_event.is_set():
            messagebox.showinfo("Recorder", "Already recording.")
            return
        outdir = make_timestamped_dir(self.parent_dir.get())
        base = self.base_name.get() or f"recording_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        safe_base = "".join(c if c not in r'<>:"/\|?*' else "_" for c in base).strip() or "recording"
        video_path = os.path.join(outdir, safe_base + ".mp4")
        audio_path = os.path.join(outdir, safe_base + ".wav")
        self.latest_video = video_path
        self.latest_audio = audio_path
        self.latest_muxed = None

        self.stop_event = threading.Event()
        t_vid = threading.Thread(target=video_recorder_stream,args=(self.stop_event, video_path, self.camera.get(), self.fps.get(), self.show_preview.get()), daemon=True)
        t_audio = threading.Thread(target=audio_recorder_stream,args=(self.stop_event, audio_path), kwargs={'channels':1}, daemon=True)
        self.threads = [t_vid, t_audio]
        t_vid.start()
        t_audio.start()
        if SR_AVAILABLE:
            t_voice = threading.Thread(target=voice_stop_listener, args=(self.stop_event, self.phrase.get()), daemon=True)
            t_voice.start()
            self.threads.append(t_voice)

        self.status.set(f"Recording... saving in: {outdir}")
        self._update_menu_state_recording()

    def stop_recording(self):
        if not self.stop_event:
            return
        self.stop_event.set()
        for t in self.threads:
            t.join(timeout=3)

        self.status.set("Finalizing...")
        self.root.update_idletasks()

        def wait_for_file(path, timeout=3.0):
            end = time.time() + timeout
            while time.time() < end:
                if path and os.path.exists(path) and os.path.getsize(path) > 0:
                    return True
                time.sleep(0.1)
            return False

        if not wait_for_file(self.latest_video, timeout=3.0):
            messagebox.showwarning("Recorder", f"Video file missing or empty: {self.latest_video}")
            self._update_menu_state_idle()
            return
        if not wait_for_file(self.latest_audio, timeout=3.0):
            messagebox.showwarning("Recorder", f"Audio file missing or empty: {self.latest_audio}")
            self._update_menu_state_idle()
            return

        ff = find_ffmpeg()
        if not ff:
            messagebox.showwarning("Recorder", "ffmpeg not found. Install ffmpeg or set ffmpeg path in Options.")
            self._update_menu_state_idle()
            return

        out_mux = os.path.splitext(self.latest_video)[0] + "_with_audio.mp4"
        cmd = [
            ff, "-y",
            "-i", self.latest_video,
            "-i", self.latest_audio,
            "-c:v", "copy",
            "-c:a", "aac",
            "-b:a", "192k",
            "-map", "0:v:0",
            "-map", "1:a:0"
        ]
        if self.shortest.get():
            cmd.append("-shortest")
        cmd.append(out_mux)

        try:
            proc = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, check=False)
            if proc.returncode != 0:
                err = proc.stderr.decode(errors="ignore")
                messagebox.showerror("Mux failed", f"ffmpeg returned code {proc.returncode}\n\n{err}")
                self._update_menu_state_idle()
                return
            else:
                self.latest_muxed = out_mux
                self.status.set(f"Idle — muxed: {out_mux}")
                messagebox.showinfo("Recorder", f"Recording stopped and saved.\nMuxed file:\n{out_mux}")
                self.recordings.insert(0, (os.path.dirname(self.latest_video), self.latest_video, self.latest_audio, self.latest_muxed))
                self._refresh_recordings_list()
        except Exception as e:
            messagebox.showerror("Mux error", str(e))

        self._update_menu_state_idle()

    def _refresh_recordings_list(self):
        self.listbox.delete(0, tk.END)
        for idx, (d, v, a, m) in enumerate(self.recordings):
            name = os.path.basename(v)
            display = f"{idx+1}. {name} — {os.path.basename(m) if m else '(no mux)'}"
            self.listbox.insert(tk.END, display)

    def get_selected_recording(self):
        sel = self.listbox.curselection()
        if not sel:
            if not self.recordings:
                return None
            return self.recordings[0]
        return self.recordings[sel[0]]

    def open_selected(self):
        rec = self.get_selected_recording()
        if not rec:
            messagebox.showinfo("Open", "No recording selected.")
            return
        _, video, audio, muxed = rec
        if muxed and os.path.exists(muxed):
            play_file(muxed)
        else:
            play_file(video)

    def open_selected_folder(self):
        rec = self.get_selected_recording()
        if not rec:
            open_path_in_explorer(self.parent_dir.get())
            return
        d, _, _, _ = rec
        open_path_in_explorer(d)

    def play_selected_muxed(self):
        rec = self.get_selected_recording()
        if not rec:
            messagebox.showinfo("Play", "No recording selected.")
            return
        _, _, _, muxed = rec
        if muxed and os.path.exists(muxed):
            play_file(muxed)
        else:
            messagebox.showwarning("Play", "Muxed file not available for this recording.")

    # --- Transcription helpers ---

    def transcribe_selected_muxed(self):
        if not SR_AVAILABLE:
            messagebox.showwarning("Transcribe", "speech_recognition not available.")
            return
        rec = self.get_selected_recording()
        if not rec:
            messagebox.showinfo("Transcribe", "No recording selected.")
            return
        _, _, _, muxed = rec
        if not muxed or not os.path.exists(muxed):
            messagebox.showwarning("Transcribe", "Muxed video not available for this recording.")
            return
        self._transcribe_from_video(muxed)

    def transcribe_last_muxed(self):
        if not SR_AVAILABLE:
            messagebox.showwarning("Transcribe", "speech_recognition not available.")
            return
        if not self.latest_muxed or not os.path.exists(self.latest_muxed):
            messagebox.showwarning("Transcribe", "No recent muxed MP4 available.")
            return
        self._transcribe_from_video(self.latest_muxed)

    def transcribe_last_wav(self):
        if not SR_AVAILABLE:
            messagebox.showwarning("Transcribe", "speech_recognition not available.")
            return
        if not self.latest_audio or not os.path.exists(self.latest_audio):
            messagebox.showwarning("Transcribe", "No recent WAV available.")
            return
        self._do_transcription(self.latest_audio)

    def _transcribe_from_video(self, video_path):
        ff = find_ffmpeg()
        if not ff:
            messagebox.showwarning("Transcribe", "ffmpeg not found. Install ffmpeg or set ffmpeg path in Options.")
            return
        # extract audio to temporary wav next to video
        tmp_wav = os.path.splitext(video_path)[0] + "_extracted.wav"
        cmd = [ff, "-y", "-i", video_path, "-vn", "-acodec", "pcm_s16le", "-ar", "16000", "-ac", "1", tmp_wav]
        try:
            proc = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, check=False)
            if proc.returncode != 0 or not os.path.exists(tmp_wav):
                err = proc.stderr.decode(errors="ignore")
                messagebox.showerror("Extract audio failed", f"ffmpeg returned code {proc.returncode}\n\n{err}")
                return
            # Run transcription on extracted WAV
            self._do_transcription(tmp_wav)
        except Exception as e:
            messagebox.showerror("Transcription error", str(e))

    def _do_transcription(self, wav_path):
        self.status.set("Transcribing...")
        self.root.update_idletasks()
        try:
            r = sr.Recognizer()
            with sr.AudioFile(wav_path) as source:
                audio = r.record(source)
            # Use Google's free recognizer (internet required)
            text = r.recognize_google(audio)
            out_txt = os.path.splitext(wav_path)[0] + ".txt"
            with open(out_txt, "w", encoding="utf-8") as f:
                f.write(text)
            messagebox.showinfo("Transcription", f"Transcription finished and saved to:\n{out_txt}\n\nFirst 400 chars:\n{text[:400]}")
            self.status.set("Idle — transcription saved")
        except sr.UnknownValueError:
            messagebox.showwarning("Transcription", "Could not understand audio.")
            self.status.set("Idle — transcription failed")
        except Exception as e:
            messagebox.showerror("Transcription error", str(e))
            self.status.set("Idle — transcription error")
        finally:
            self.root.update_idletasks()

if __name__ == "__main__":
    root = tk.Tk()
    app = RecorderApp(root)
    root.mainloop()
