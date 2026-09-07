# meetcap

Record a meeting and transcribe it locally. Nothing leaves the machine unless
you send it somewhere yourself.

## The app

    meetcap-app

Recording controls with input *and* output device pickers and a mic test,
your recordings in a list, a model picker, playback with seek so you can
check a recording, and the transcript readable beside it. It drives the
command-line tools below rather than reimplementing them, so the capture path
is the one that has been tested against real hardware.

**Listen to** picks a side: *Both* is the conversation, mixed on demand and
cached under `~/.cache/meetcap`, while *You* and *Meeting* are the raw tracks
as recorded. The recordings themselves are never modified.

Every recording in the list carries its own **Play** and **PDF** buttons, so
reviewing one is a click on its row rather than a hunt through a folder. While
the audio plays the transcript follows it, highlighting what is being said and
scrolling only when the highlight would otherwise leave the screen - word by
word when the transcript has word timings, and by phrase when it does not, so
transcripts made before those timings existed still follow along.

Needs PyQt6. Everything below works without it.

## One command for a whole meeting

    meetcap-meeting faculty-sync

Records until you press Ctrl-C, then transcribes and shows the transcript.
Stopping the recording is what starts the transcription - nothing after the
meeting needs typing.

## The pieces, if you want them separately

    meetcap rec --app zoom --name "faculty-sync"
    meetcap-transcribe run ~/Recordings/meetings/<file>-mic.wav

## Why two tracks

`meetcap` records **you** and **everyone else** as separate files:

    2026-09-04_1000_faculty-sync-mic.wav      your microphone
    2026-09-04_1000_faculty-sync-remote.wav   the meeting's audio
    2026-09-04_1000_faculty-sync.json         metadata + transcript links

Two independent ffmpeg processes, deliberately: if one dies mid-meeting the
other keeps recording. Keeping the streams apart also means the transcript
knows who spoke without any diarization model.

They are WAV, not FLAC, also deliberately. FLAC buffers - a killed recorder
leaves a zero-byte file and the meeting is simply gone. WAV is written
continuously, so a crash still leaves readable audio, and the growing byte
count during recording is honest proof that audio is flowing. Roughly 230 MB
per hour for the pair at the default 16 kHz; `--compress` shrinks that about
tenfold to Opus after a clean stop, or `--compress flac` to stay lossless.

## Recording

    meetcap list                      # which apps are playing audio
    meetcap mics                      # which microphones exist
    meetcap check --app zoom          # prove both halves work BEFORE the meeting
    meetcap rec --follow zoom         # the device zoom is using (safest)
    meetcap rec --app zoom            # ONLY zoom, isolated (best separation)
    meetcap rec                       # the whole system mix

Three capture modes, in order of increasing risk:

`--follow APP` finds the output device the app is actually playing to and
records that device's monitor. It reroutes nothing at all, so it cannot affect
what you hear, and unlike the plain system mix it cannot miss a meeting that is
playing to a device other than your default output. **Use this for a meeting
that matters.**

`--app APP` isolates hardest - only that app lands in the recording - but it
does so by moving the app's audio into a private sink and looping it back to
your speakers. That loopback is a dependency, and the failure mode is an hour
spent unable to hear anyone. Worth using once you have watched it behave.

Plain `meetcap rec` records your default output. Simple, but wrong if the
meeting is not playing there.

`--app` takes any fragment of the application name or its binary, as shown by
`meetcap list`. It routes that app into a private PipeWire sink and records
only that, looping the audio back to your speakers so you still hear the
meeting. New streams the app opens mid-call (a screen share, a second device)
are picked up automatically. Everything is unwound on exit.

When it starts, it prints **where you will hear the app** - capturing an app
moves its audio out of your speakers and the loopback is what puts it back, so
that line is worth reading. `--monitor-sink` sends it somewhere else if your
default output is unreliable. If the loopback cannot be created it says so
loudly rather than leaving you deaf for an hour.

Omit both flags and you get the whole system mix. It reroutes nothing either,
but it records your *default* output, so it misses a meeting playing anywhere
else - which is exactly what `--follow` was added to fix. Reach for it only
when you know the meeting is on your default device.

Stop with Ctrl-C.

## Transcribing

    meetcap-transcribe probe          # what THIS machine can run
    meetcap-transcribe run <file>     # auto-picks a model that fits
    meetcap-transcribe run <file> --model large-v3
    meetcap-transcribe run <file> --remote todd@gpu-box

Nothing is hardcoded to a particular machine. `probe` looks at the actual GPU,
VRAM, RAM, and which whisper implementations are installed, then recommends the
largest model that both **fits in memory and finishes in reasonable time** -
those are different limits. `large-v3` fits in 8 GB of RAM but runs slower than
realtime on a CPU, so an hour of meeting would take over three hours. `probe`
tells you that up front instead of letting you find out afterwards.

Ask for a model that doesn't fit and it says so rather than dying halfway
through. `--force` overrides.

`--remote` copies the audio to another machine over ssh, transcribes there, and
brings the transcript back - so you can record on whatever box is in the
meeting and transcribe on whatever box has the GPU.

Output is `<recording>-transcript.md` (readable, speaker-attributed) and
`-transcript.json` (segments with timestamps, and per-word timings where the
backend provides them - that is what lets the app follow the transcript word by
word during playback).

## Hardware

**Recording asks almost nothing.** Two ffmpeg processes writing 16 kHz mono;
any machine that can run the meeting can record it. Budget ~230 MB of disk per
hour for the pair, and a working microphone.

**Transcribing is where the hardware shows.** Both tracks are transcribed, so
a one-hour meeting is two hours of audio. Times below are for the whole
meeting, and memory is what the model needs while running:

| Model | Memory | 1-hour meeting, GPU | 1-hour meeting, CPU |
|---|---|---|---|
| `tiny` | ~1 GB | ~2 min | ~8 min |
| `base` | ~1.2 GB | ~3 min | ~12 min |
| `small` | ~2.2 GB | ~4 min | ~30 min |
| `medium` | ~5 GB | ~7 min | ~80 min |
| `large-v3` | ~10 GB | ~12 min | ~200 min |

GPU figures assume CUDA in float16; an RTX 5090 does a one-hour meeting on
`large-v3` in about 12 minutes. Apple Silicon with whisper.cpp and Metal lands
between the two columns.

Fitting in memory and finishing in reasonable time are different limits, and
the second is the one that bites: `large-v3` fits in 8 GB of RAM but runs
slower than realtime on a CPU, so an hour of meeting takes over three hours.
`meetcap-transcribe probe` measures *your* machine and picks accordingly - the
table is a guide, not a hardcoded assumption.

No GPU? `small` on a CPU is the sweet spot, and `--remote` sends the audio to a
machine that has one.

## Requirements

Recording needs only `ffmpeg` and `pactl` (PipeWire or PulseAudio). No Python
packages at all - that is the point, so it cannot fail to start.

Transcribing needs one whisper backend; `probe` names one that suits the
machine. `faster-whisper` is the usual choice and picks up CUDA automatically.

If PyAV has no wheel for your Python version (3.14, currently), install with
`pip install --no-deps faster-whisper` plus `ctranslate2 tokenizers
onnxruntime huggingface_hub tqdm`. meetcap decodes audio with the ffmpeg it
already requires and stubs PyAV out, so it is not actually needed.

## Before a meeting you care about

    tools/selftest                    # proves the whole chain, end to end
    meetcap mics                      # confirm a real mic exists
    meetcap check --app zoom          # both lines must say OK

`tools/selftest` plays a synthetic colleague through your speakers as a
capturable app, records it alongside your microphone, transcribes both, and
tells you which halves worked. Speak when it prompts you. If it prints your
words under **You** and the colleague's under **Meeting**, everything works:
microphone, per-app isolation, two-track separation, and transcription.

`check` is the whole point of this being a separate step. A meeting recorded
with a dead microphone looks exactly like a working one until you play it back.

## Note

Tell people you are recording. In some places it is also the law.
