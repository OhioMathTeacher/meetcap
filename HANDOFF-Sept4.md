# Handoff — 4 September 2026

Built overnight, 3–4 September. `meetcap` records a meeting locally and
transcribes it with Whisper, so you get notes without handing the audio to
anyone's cloud.

Repo: <https://github.com/OhioMathTeacher/meetcap> (private)
At `06a3b54`. Both machines are clean clones of that commit.

---

## Do this before the meeting

**Switch on the AT2020USB-X.** It is the default microphone on
todd-gpt-fedora and it delivers pure digital silence — measured repeatedly at
−91.0 dB, which is digital zero, not a quiet room. It enumerates on USB, is
not muted in software, and sits at +10.9 dB gain, so nothing in software will
tell you it is off. Almost certainly the touch-mute on the mic body.

Then, in the office:

    cd ~/Repos/meetcap && git pull      # will say up to date
    tools/selftest                      # speak when prompted; want both PASS

`selftest` plays a synthetic colleague through the speakers as a real
capturable app, records it against the live mic, transcribes both tracks and
reports PASS/FAIL per half. It is the only thing that proves the whole chain
on a given machine.

Fallback if the AT2020 stays dead — both of these are live on that box:

    --mic alsa_input.usb-Generic_NexiGo_N660P_FHD_Webcam_200901010001-02.analog-stereo
    --mic alsa_input.pci-0000_80_1f.3.analog-stereo

## Recording the meeting

Either open the app:

    meetcap-app

or one command that records, then transcribes when you stop it:

    meetcap-meeting colleagues

or the pieces:

    meetcap rec --follow zoom --name colleagues     # Ctrl-C to stop
    meetcap-transcribe run ~/Recordings/meetings/<file>-mic.wav

**Use `--follow`, not `--app`, for a meeting that matters.** It records the
output device Zoom is using and reroutes nothing, so it cannot affect what you
hear. `--app` isolates better but moves Zoom's audio through a private sink and
depends on a loopback; if that fails you spend an hour unable to hear anyone.

---

## What is here

    bin/meetcap             record (list, mics, check, rec)
    bin/meetcap-transcribe  transcribe (probe, run)
    bin/meetcap-app         the GUI
    bin/meetcap-meeting     record → transcribe → show, one command
    bin/meetcap-commands    every command with Copy buttons
    tools/selftest          prove the chain end to end

Recording needs only ffmpeg and pactl — no Python packages, so it cannot fail
to start. Only transcription needs a whisper backend.

Everything for one recording lands in the same folder, `~/Recordings/meetings`
by default:

    <stamp>_<name>-mic.wav          you
    <stamp>_<name>-remote.wav       the meeting
    <stamp>_<name>.json             metadata
    <stamp>_<name>-transcript.md    readable, speaker-attributed
    <stamp>_<name>-transcript.json  segments with timestamps

## Machine state

**todd-gpt-fedora** — RTX 5090 (32 GB), 64 GB RAM, Fedora 43. Zoom is an RPM.
Whisper venv on **python3.12** at `~/Repos/meetcap/.venv`; all five commands
symlinked into `~/.local/bin`. `large-v3` on CUDA does a 1-hour meeting in
~12 minutes. Default output is the (Office) Esinkin BT Adapter.

**imac-fedora** — no CUDA, so `auto` picks `small` (~30 min/hour). No built-in
microphone at all; a USB camera provides one. PyQt6 is not installed
system-wide here — the GUI runs from streamcapture's venv.

---

## Decisions worth not re-litigating

**Two separate tracks, two ffmpeg processes.** If one dies the other keeps
recording, and separating you from everyone else gives speaker attribution
without a diarization model.

**WAV during capture, compression only after.** Measured, not assumed. Killed
with SIGKILL mid-recording: WAV returns everything written so far; FLAC, Opus
and MP3 each leave exactly zero bytes. `--compress` runs after a clean stop.

**16 kHz.** Whisper resamples to 16 kHz regardless, so 48 kHz stored three
times the data for a byte-identical transcript. ~230 MB/hour for the pair,
~20 MB after `--compress`. `--rate 48000` for archival audio.

**Transcription is never hardcoded to a machine.** The probe reads GPU, VRAM
and RAM at runtime and recommends the largest model that both fits *and* beats
realtime — different limits. `large-v3` fits in 8 GB of RAM but runs slower
than realtime on a CPU.

## Things that will bite you if forgotten

- **A monitor tap sits after the volume control.** Muting your speakers records
  digital silence from everyone else (measured: −18 dB at 100%, −49 dB at 30%,
  −91 dB muted). The recorder warns; the app shows it live.
- **Whisper hallucinates sentences out of near-silence.** `large-v3` produced
  "This is the city." from an empty room. A transcript appearing is not proof
  a microphone worked — the selftest's PASS lines are.
- **Python 3.14 has no PyAV wheel**, so `pip install faster-whisper` fails.
  Install `--no-deps`; meetcap decodes with ffmpeg and stubs PyAV out.
- **CTranslate2 needs `nvidia-cublas-cu12` and `nvidia-cudnn-cu12`** and cannot
  find them without the in-process preload in `meetcap-transcribe`.
- **Resuming a suspended PipeWire source emits a startup pop** that reads as
  full-scale audio. Level checks discard the first second; without that, a dead
  mic tests healthy.

## Open

- **macOS port.** The transcriber is portable today (whisper.cpp + Metal, and
  the probe knows about Apple Silicon). The recorder is not: it is entirely
  PipeWire/pactl, and macOS cannot capture system audio at all without a
  virtual device such as BlackHole. Per-app capture has no macOS equivalent
  reachable from ffmpeg. No macOS machine here is set up for testing yet, so
  none of this has been tried in practice.
- **No engine picker in the app.** Model size is selectable; faster-whisper vs
  whisper.cpp vs `--remote <host>` is auto-detected or CLI-only. Sending audio
  from a laptop to the 5090 from inside the app is the obvious next feature.
- **No delete in the app.** Recordings can only be removed from the folder.
- The GUI has never run on a real X session on the GPU box — only headless
  screenshots. It has run for real on imac-fedora.
