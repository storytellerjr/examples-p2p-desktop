# basic-live-cam — possible future improvements

This file collects forward-looking ideas surfaced while studying this folder. It is intentionally separate from `notes.md` (Q&A study notes) so the two don't get tangled. Each entry is anchored to code with `path:line` and explains *what*, *why*, and *rough how*.

## 1. Capture and transmit audio alongside the video

**What.** The app advertises itself as a "live cam" and even prompts for microphone permission (`ui/root.jsx:18-20`), but no audio ever leaves the broadcaster. Three places make the pipeline video-only:

- FFmpeg input selects video devices only (`worker/live-cam-room.js:196-214`):
  - macOS: `-f avfoundation -i 0` — index `0` is the first video device. Audio needs the dual form `-i "0:0"` (video:audio).
  - Linux: `-f v4l2 -i /dev/video0` — v4l2 has no audio path.
  - Windows: `-f dshow -i 'video=Integrated Camera'` — no `audio=...` companion.
- FFmpeg output declares only a video codec (`worker/live-cam-room.js:215-223`): `-c:v libx264 …` with no `-c:a` audio codec.
- The MediaSource SourceBuffer on the receiver is created with `'video/mp4; codecs="avc1.640028"'` only (`ui/root.jsx:34`) — H.264 video, no `mp4a.40.2` (AAC) audio track. Even if FFmpeg muxed audio, the browser would reject the buffer.

**Why.** A "live cam" with no sound is a surprising and substantial UX gap for anyone using this as a reference implementation. The mic-permission prompt actively misleads — the user is asked to grant a permission the app never uses. For real use cases (remote presence, a Twitch/Instagram-Live-style room, a P2P "call"), audio is table-stakes, not nice-to-have.

**Rough how.**

- **Capture.** Add audio inputs per platform:
  - macOS: `-f avfoundation -framerate 30 -i "0:0"` (first video device + first audio device).
  - Linux: keep `-f v4l2 -i /dev/video0` for video and add a second input `-f alsa -i default` (or pulse) for audio; FFmpeg supports multiple `-i` flags.
  - Windows: add `-f dshow -i audio="Microphone (…)"` alongside the video dshow input.
- **Encode.** Add `-c:a aac -b:a 128k -ar 48000` to the output args. Keep the fragmented-MP4 muxing flags as-is — fragmented MP4 supports interleaved AAC + H.264.
- **Receive.** Replace the single video-only SourceBuffer with the combined codec string `'video/mp4; codecs="avc1.640028, mp4a.40.2"'`, or attach a second SourceBuffer for the audio track if the demux needs splitting. Verify with `MediaSource.isTypeSupported(...)` before adding.
- **UI.** Add a mute toggle and a volume slider next to the `<video>` element (`ui/root.jsx:64`). Default behaviour: autoplay starts muted (browsers block audible autoplay) and the user clicks to unmute, matching Twitch/YouTube conventions.

A useful staging step is to first prove an audio-only fragmented MP4 round-trip (no video), so any issues are isolated to one track at a time.

## 2. Device pickers for camera and microphone in the UI

**What.** Both inputs are currently hard-coded in the FFmpeg invocation (`worker/live-cam-room.js:196-214`). The macOS path uses `-i 0`, Linux is pinned to `/dev/video0`, Windows is pinned to `video=Integrated Camera`. There's no way for the broadcaster to choose between, say, an internal FaceTime camera and an external USB webcam, or between the laptop mic and a USB podcast mic.

**Why.**

- A user with multiple cameras (laptop + external, or laptop + iPhone Continuity Camera) can't pick which one to broadcast.
- A user whose Windows camera isn't literally named `Integrated Camera` won't broadcast anything at all — FFmpeg will fail to open the device. The comment in `worker/live-cam-room.js:212` (`// adjust device name`) acknowledges this is fragile.
- Once audio is added (feature 1), the same problem appears for microphones — most setups have at least two (laptop built-in + headset/USB mic), and the "right" default is genuinely unknowable.

**Rough how.**

- **Enumerate.** On worker startup, list available devices per platform and send the list to the UI via HRPC:
  - macOS: `ffmpeg -f avfoundation -list_devices true -i ""` (parse stderr — avfoundation prints the device list there).
  - Linux: read `/sys/class/video4linux/` and `/proc/asound/cards`, or shell out to `v4l2-ctl --list-devices` and `arecord -l`.
  - Windows: `ffmpeg -f dshow -list_devices true -i dummy`.
- **Expose via HRPC.** Add a `devices` method to `spec/hrpc` that returns `{ cameras: [{ id, label }], microphones: [{ id, label }] }`. Call it once when the worker opens; optionally refresh on demand.
- **Render dropdowns.** Two `<select>` elements above the `<video>` in `ui/root.jsx`, populated from the device list. Persist the last-chosen IDs in localStorage so a returning broadcaster keeps their selection.
- **Apply selection.** When the user picks a device, send the chosen ID via a new `setDevices(cameraId, micId)` HRPC call. The worker stops the current FFmpeg (`worker/live-cam-room.js:107` `this.ffmpeg?.kill('SIGKILL')`) and respawns with the new `-i …` arguments.
- **Defaults.** First start: pick the first camera + first mic the OS reports, instead of the hard-coded `0` / `/dev/video0` / `'Integrated Camera'`. That alone fixes the Windows brittleness without any UI work.

The dropdowns are broadcaster-only — they make no sense on the joining peers (`if (!this.invite) this._startLiveCam()` at `worker/live-cam-room.js:103`). The UI should gate the controls on "am I the broadcaster" so joiners don't see useless device pickers next to their playback view.

## 3. Let the broadcaster share their screen instead of (or alongside) the camera

**What.** The broadcaster (the peer that ran `pear run` without `--invite` — gated by `if (!this.invite) this._startLiveCam()` at `worker/live-cam-room.js:103`) can only broadcast their webcam. There is no way to share a desktop, a single application window, or a second monitor. The FFmpeg input is wired strictly to camera-capture APIs (`worker/live-cam-room.js:197-214`).

**Why.** Screen-share is the canonical use case for "live cam"-style desktop apps: pair-programming, remote support, "look at this thing on my screen" calls, sharing slides during a presentation. Without it, this example caps out as a casual face-cam stream. Compared to feature 1 (audio) and feature 2 (device pickers), screen-share is the highest-leverage addition for turning this into a useful demo of P2P live media.

**Rough how.**

- **Capture source.** Swap the FFmpeg input flags per platform when "share screen" is selected:
  - macOS: avfoundation lists screens alongside cameras. `ffmpeg -f avfoundation -list_devices true -i ""` exposes entries like `Capture screen 0`, `Capture screen 1`. Use `-f avfoundation -framerate 30 -i "<screen-index>"` (e.g. `-i "2"` if screen 0 is the third device). macOS will trigger its own Screen Recording permission prompt the first time — the current `ui.media.access.camera()` call (`ui/root.jsx:14-21`) does not cover this; the app will need to handle the prompt (or surface a "grant Screen Recording in System Settings → Privacy" hint when capture fails).
  - Linux (X11): `-f x11grab -framerate 30 -i :0.0` for the whole primary display, or `-i :0.0+x,y -video_size WxH` for a region. Wayland needs pipewire + the XDG screen-cast portal — out of scope for a first cut; document it as "X11 only".
  - Windows: `-f gdigrab -framerate 30 -i desktop` for the whole desktop, or `-i title="<window title>"` to capture a single window.
- **Source selector.** Extend the device picker from feature 2 to a three-option control: *Camera* / *Screen* / *Window*. When *Screen* or *Window* is selected, populate a secondary dropdown with the available displays / window titles (parsed from the same FFmpeg `-list_devices` output on macOS, from `xrandr --listmonitors` and `wmctrl -l` on Linux, from `EnumWindows` via a small helper on Windows).
- **Switching mid-broadcast.** Reuse the respawn pattern from feature 2 — `this.ffmpeg?.kill('SIGKILL')` (`worker/live-cam-room.js:107`) and start a fresh FFmpeg with the new input flags. **Caveat:** every fragment ever produced is appended to the autobase view (`worker/live-cam-room.js:268-272`) and replayed in order by joiners (`ui/root.jsx:30-49`), so switching from camera to screen produces a single MP4 stream with a hard resolution/format change mid-playback. The browser's MediaSource may refuse to append the new fragments because the SourceBuffer was initialised with the camera's `avc1` profile. Two options:
  - **Pragmatic:** reset `this.fragIdx`, start a new blob, and have the UI tear down + recreate the `MediaSource` on a source change (cheap, visible to viewers as "stream restarted").
  - **Better:** include the codec init segment per source switch and re-`addSourceBuffer` on the receiving end — more code, smoother UX.
- **Audio handling.** When pairing with feature 1, decide whether *Share Screen* also captures system audio (loopback — different per OS: `dshow audio="virtual-audio-capturer"` on Windows, BlackHole/Loopback on macOS, `pulse` monitor source on Linux) or stays on the microphone. Most apps offer both as separate toggles ("share screen audio", "share microphone") and that's a sensible default here.
- **Privacy warning.** Screen-share has obvious leak potential — notifications, password managers, other browser tabs. Add a confirm-before-broadcast dialog the first time the user selects *Screen*, similar to Zoom's "everyone will see this preview" step, plus a persistent banner in the UI while screen-share is live so the broadcaster can't forget the stream is going.

Like the device pickers, the screen-share controls are broadcaster-only. Joiners just receive whatever fragments arrive — the receiving code in `ui/root.jsx:30-49` doesn't care whether the bytes started life as a camera or a screen.

## 4. Quality / latency presets to reduce playback lag

**What.** A noticeable delay (typically several seconds) between what the broadcaster does and what the viewer sees, plus occasional stalls. The current FFmpeg pipeline ships at native camera resolution and unconstrained bitrate, with a 2-second-GOP fragment boundary (`worker/live-cam-room.js:197-223`):

- `-framerate 30` at the input, no `-video_size` cap → the encoder runs at the camera's native resolution (often 1280×720 or 1920×1080).
- `-preset ultrafast -tune zerolatency` is already good on the encoder side.
- `-g 60` → one keyframe every 60 frames = every 2 seconds at 30fps.
- `-movflags frag_keyframe+empty_moov+default_base_moof` → each MP4 fragment ends at a keyframe boundary, i.e. each fragment is one full GOP ≈ 2 seconds of video.
- No `-b:v` / `-maxrate` / `-bufsize`, so the encoder is free to emit large keyframes.

Combined with the replication path (each fragment is written to `Hyperblobs` at `worker/live-cam-room.js:261-265`, appended to the autobase view at `:268-272`, fetched over `hypercore-blob-server` by the viewer at `ui/root.jsx:42-46`), the structural latency floor is roughly: *time to fill one GOP* + *time to replicate the fragment over the swarm* + *MediaSource append + decode*. The first term alone is ~2 s by configuration.

**Why.** For face-cam and screen-share interactions, multi-second lag breaks the feel — comments arrive before the reaction, screen pointers feel disconnected from the broadcaster's voice (once audio lands — feature 1). The example is impressive as a P2P proof-of-concept but feels broken as a "live" app. Letting the broadcaster trade resolution / framerate for responsiveness fixes both the perceived lag and the bandwidth burst on slow links.

**Rough how.**

- **Add a quality preset selector** to the broadcaster UI (`ui/root.jsx`), next to the device pickers from feature 2. Three presets is enough:
  - *Low / responsive* — 480×270, 15 fps, `-b:v 400k -maxrate 600k -bufsize 800k`, `-g 15` (1 s keyframe interval). Fragments ≈ 1 s each. Best for chat-style usage on flaky networks.
  - *Medium* — 640×360, 24 fps, `-b:v 800k -maxrate 1.2M -bufsize 1.6M`, `-g 24`. Default.
  - *High* — current behaviour (native res, 30 fps, `-g 60`). Lowest latency-floor only when bandwidth is abundant on both sides.
- **Wire the preset into FFmpeg.** In `_startLiveCam` (`worker/live-cam-room.js:196`) replace the hard-coded `FF_INPUT` and `FF_OUTPUT` arrays with values built from the active preset. Add `-video_size WxH` to the input (avfoundation/v4l2/dshow all accept it). Add `-b:v`, `-maxrate`, `-bufsize`, and a smaller `-g` to the output.
- **Switch live.** Reuse the FFmpeg respawn pattern from features 2 and 3 (`this.ffmpeg?.kill('SIGKILL')` at `worker/live-cam-room.js:107`). Same MediaSource caveat applies — resolution changes mid-stream may require a SourceBuffer rebuild; the simplest path is to tear down and recreate the `MediaSource` on the viewer side when a new "init segment" arrives. Mark the autobase row with a `presetVersion` field so the viewer can detect the change in `ui/root.jsx:36-49`.
- **Persist the choice.** Save the last-used preset in localStorage so a returning broadcaster keeps their setting, mirroring the device-picker persistence from feature 2.

**Caveats — what this won't fix.**

- The GOP-sized fragment boundary is a structural latency floor. Even at *Low / responsive* the viewer can't start playback until at least one full fragment (≈ 1 s) has been replicated. Reducing `-g` below ~10 frames raises bitrate sharply (more keyframes) for diminishing latency returns.
- The viewer's playback loop polls every 100 ms for the next fragment (`ui/root.jsx:38-40`). On a fast link that's fine, but bursty fragment arrival can stall playback briefly. A `setInterval`-driven push from `useWorker` (`lib/use-worker.js`) would be tighter but is a separate refactor.
- Cross-peer clock and bandwidth heterogeneity means joiners on slow links will lag joiners on fast links — there is no adaptive-bitrate ladder here (and adding one is much bigger than a preset switch).
