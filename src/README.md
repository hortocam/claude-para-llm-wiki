# Python Utilities — Setup Guide

The `src/` directory contains Python utilities used by the claude-para-llm-wiki plugin.
These scripts are invoked by hooks and commands — not directly by the LLM.

## Requirements

Python 3.10+ is required.

## Setup

### macOS (Apple Silicon — recommended path)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r src/requirements.txt

# For Apple Silicon transcription acceleration (optional but recommended):
pip install mlx-whisper
```

### macOS (Intel) / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r src/requirements.txt
```

### Windows (Git Bash or WSL recommended)

```bash
python -m venv .venv
source .venv/Scripts/activate   # Git Bash
# OR: .venv\Scripts\activate.bat  # CMD
pip install -r src/requirements.txt
```

## Transcription Backend Selection

The plugin auto-selects the transcription backend at runtime:

| Platform | Backend | Notes |
|----------|---------|-------|
| Apple Silicon Mac | mlx-whisper | Fastest; requires `pip install mlx-whisper` |
| Mac Intel / Linux / Windows | faster-whisper | CPU/CUDA auto-detect; slower |

Detection logic in `transcribe.py`:
```python
import sys, torch
use_mlx = sys.platform == "darwin" and torch.backends.mps.is_available()
```

## Model Size Guidance

| Model | Speed | Quality | Use for |
|-------|-------|---------|---------|
| `base` | Fast | Good | Clear speech, meeting notes |
| `small` | Medium | Better | General use |
| `medium` | Slow | Best | Important recordings |
| `large-v3` | Slowest | Maximum | Apple Silicon / GPU only |

Default: `base`. Override with `--model medium` flag.

## Dependency Notes

- **llm-guard**: ProtectAI's PromptInjectionScanner. Requires network on first run
  to download model weights (~50MB). Weights are cached locally after first run.
- **trafilatura**: Best-in-class web content extraction. Primary HTML→Markdown converter.
- **beautifulsoup4 / lxml**: Fallback extraction for sites trafilatura misses.
- **faster-whisper**: CTranslate2-based Whisper. Much faster than openai-whisper on CPU.
- **torch**: Required for platform detection (`mps.is_available()`). CPU-only install
  is fine; GPU is not required.
