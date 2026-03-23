# Plaud Meeting Summarizer

Automatically transcribes Plaud voice recordings and generates structured meeting notes as Word documents using local Whisper (free) and the Claude API.

## What it does

1. Watches your iCloud `PlaudInput` folder for new recordings
2. Transcribes audio locally using OpenAI Whisper (runs on your machine, no cost)
3. Summarizes with Claude and structures the output into:
   - Attendees
   - Summary (TL;DR)
   - Key Decisions
   - Action Items (table with owner + deadline)
   - Discussion Topics
   - Open Questions
4. Saves a formatted `.docx` — prompts you to choose the save location via a native macOS folder picker

## Requirements

- Python 3.10+
- An [Anthropic API key](https://console.anthropic.com/)
- A Plaud device syncing recordings to iCloud

## Setup

**1. Clone and create a virtual environment**
```bash
git clone <repo-url>
cd plaud-summarizer
python3 -m venv .venv
source .venv/bin/activate
```

**2. Install dependencies**
```bash
pip install openai-whisper anthropic python-docx
pip install torch torchvision torchaudio
```

**3. Set your Anthropic API key**
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

To make this permanent, add it to your `~/.zshrc` or `~/.bash_profile`.

**4. Set up iCloud folders**

The script watches:
- **Input:** `~/Library/Mobile Documents/com~apple~CloudDocs/PlaudInput`
- **Default output:** `~/Library/Mobile Documents/com~apple~CloudDocs/PlaudOutput`

On your Plaud device, set the export destination to the `PlaudInput` iCloud folder. The script creates both folders automatically on first run.

## Usage

**Watch mode** (auto-processes new files as they arrive):
```bash
python3 plaud_summarizer.py
```

**Single file:**
```bash
python3 plaud_summarizer.py recording.mp3
```

**Custom Whisper model or output directory:**
```bash
python3 plaud_summarizer.py --model medium --output ~/Documents/MeetingNotes recording.mp3
```

## Running automatically with macOS Automator

To have the script start automatically when you log in:

1. Open **Automator** → New Document → **Application**
2. Add a **Run Shell Script** action
3. Set shell to `/bin/zsh`, paste:
   ```sh
   '/path/to/.venv/bin/python3' '/path/to/plaud_summarizer.py'
   ```
4. Save and add to **Login Items** in System Settings

## Whisper model sizes

| Model | Speed | Quality | Recommended for |
|-------|-------|---------|-----------------|
| `tiny` | Fastest | Low | Testing only |
| `base` | Fast | Decent | Short, clear recordings |
| `small` | Moderate | Good | **Default — best balance** |
| `medium` | Slow | Very good | High accuracy needs |
| `large` | Slowest | Best | Near-perfect transcription |

The default is `small`. On Apple Silicon the script uses the MPS GPU automatically.

## Notes

- Transcription is fully local — audio never leaves your machine
- Only the transcript text is sent to the Claude API for summarization
- If no `ANTHROPIC_API_KEY` is set, the script saves a doc with no summary
- On macOS, a native folder picker lets you choose where to save each output file
