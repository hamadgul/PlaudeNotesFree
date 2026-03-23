# Plaud Meeting Summarizer

Automatically transcribes Plaud voice recordings and generates structured meeting notes as Word documents using local Whisper (free) and the Claude API.

## What it does

1. On first run, prompts you to select your input and output folders using a native macOS folder picker — remembered for all future runs
2. Watches your chosen input folder for new recordings
3. Transcribes audio locally using OpenAI Whisper (runs on your machine, no cost)
4. Summarizes with Claude and structures the output into:
   - Attendees
   - Summary (TL;DR)
   - Key Decisions
   - Action Items (table with owner + deadline)
   - Discussion Topics
   - Open Questions
5. Saves a formatted `.docx` — prompts you to choose the save location via a native macOS folder picker

## Requirements

- Python 3.10+
- An [Anthropic API key](https://console.anthropic.com/)
- A Plaud device syncing recordings to iCloud

## Setup

**1. Clone and create a virtual environment**
```bash
git clone https://github.com/hamadgul/PlaudeNotesFree.git
cd plaud-summarizer
python3 -m venv .venv
source .venv/bin/activate
```

**2. Install dependencies**
```bash
brew install ffmpeg
pip install openai-whisper anthropic python-docx
pip install torch torchvision torchaudio
```

**3. Set your Anthropic API key**
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

To make this permanent:
```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-your-key-here"' >> ~/.zshrc && source ~/.zshrc
```

**4. Run the script — folder setup happens automatically**

On first run, two native macOS folder pickers will appear:
1. **Input folder** — where your Plaud recordings are dropped (e.g. an iCloud or local folder)
2. **Output folder** — default location for saved notes

Your selections are remembered in `~/.config/plaud_notes/config.json` and used for all future runs.

To change folders at any time:
```bash
python3 plaud_summarizer.py --setup
```

## Usage

**Watch mode** (auto-processes new files as they arrive):
```bash
python3 plaud_summarizer.py
```

**Single file:**
```bash
python3 plaud_summarizer.py recording.mp3
```

**Custom Whisper model or one-off output directory:**
```bash
python3 plaud_summarizer.py --model medium --output ~/Documents/MeetingNotes recording.mp3
```

**Change your saved input/output folders:**
```bash
python3 plaud_summarizer.py --setup
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
