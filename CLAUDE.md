# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

ASCILINE — streams video as real-time ASCII/pixel art via WebSocket to a browser. Backend encodes frames, browser renders on canvas. Full A/V sync, adaptive compression, seek/pause.

## Running

```bash
# Single video
python stream_server.py video.mp4 --cols 240

# Folder queue
python stream_server.py --folder videos --cols 200 --loop

# JSON playlist (per-video overrides)
python stream_server.py --playlist playlist.json --cols 220

# Pixel mode (colored blocks, high fidelity)
python stream_server.py video.mp4 --mode 5 --pixel --cols 320

# Standalone terminal player (no web server)
python ascii_video_player2.py video.mp4 --cols 100
```

Key flags: `--mode {1-5}` (color depth), `--pixel`, `--cols`, `--rows`, `--vol {0-5}`, `--loop`, `--quality {lossless,high,balanced,low}`, `--host`, `--port`, `--debug`.

## Testing

```bash
# Cross-language codec test vectors
python experiments/gen_vectors.py

# End-to-end Python encoder ↔ JS decoder validation
node experiments/test_e2e.js <port> [maxFrames]

# Generate synthetic test clips (requires FFmpeg)
bash experiments/make_test_clips.sh
```

## Dependencies

No package.json. No requirements.txt. Install manually:

```bash
pip install fastapi uvicorn opencv-python numpy websockets
# Also: FFmpeg (winget install ffmpeg / brew install ffmpeg)
```

## Architecture

### Data Flow

```
Video → OpenCV (VideoDecoder) → AsciiMapper (char + color quantize + RLE)
      → Codec (pick smallest of RAW/ZLIB/DELTA) → WebSocket binary
      → Browser jitter buffer → A/V sync (audio master clock) → Canvas
Audio → FFmpeg → MP3 → /audio HTTP endpoint → <audio> tag
```

### Backend (`stream_server.py` + `ascii_video_player2.py` + `codec.py`)

- **`stream_server.py`**: FastAPI app. Manages video queue (single/folder/playlist), WebSocket frame streaming, audio extraction/serving, FPS decimation for >30 FPS sources, auto aspect-ratio grid sizing.
- **`ascii_video_player2.py`**: Core processing engine.
  - `VideoDecoder`: OpenCV wrapper, outputs (grayscale, BGR) pairs, supports seeking.
  - `AsciiMapper`: gray→char LUT, BGR→RGB color quantization, RLE compression.
  - `TerminalRenderer`: Standalone ANSI terminal player.
- **`codec.py`**: Adaptive per-frame codec. Encodes as RAW, ZLIB, and DELTA; picks smallest. Forces keyframes periodically. Supports lossy temporal delta (color drift tolerance).

### Frontend (`app.js` + `codec.js`)

- **`app.js`**: WebSocket client, 4-frame jitter buffer, audio master clock for A/V sync (drops stale frames, waits for future frames), pause/seek/volume, canvas renderer, invisible selection overlay for text copy.
- **`codec.js`**: Mirrors `codec.py`. Decodes RAW/ZLIB/DELTA. Runs in both browser (`DecompressionStream`) and Node.

### Codec Symmetry

`codec.py` (encoder) and `codec.js` (decoder) must stay in sync. When modifying frame format, update both. The `experiments/` tests validate this parity.

### Playlist Format

```json
[
  { "video": "intro.mp4", "mode": 1, "vol": 1 },
  { "video": "main.mp4", "mode": 5, "pixel": true, "cols": 520 }
]
```

Fields: `video`, `mode` (1-5), `pixel` (bool), `vol` (0-5), `cols`, `rows`.
