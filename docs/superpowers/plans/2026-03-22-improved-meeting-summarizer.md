# Improved Meeting Summarizer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the summarization prompt and DOCX output to produce professional, well-structured Word documents from work meeting transcripts.

**Architecture:** Claude returns a structured JSON object (attendees, tldr, decisions, action_items, topics, open_questions). A dedicated `build_docx()` function converts that JSON into a properly formatted Word doc with real tables, Word heading styles, and clean layout — no markdown parsing.

**Tech Stack:** Python 3, `anthropic` SDK, `python-docx`

---

## File Structure

- **Modify:** `plaud_summarizer.py`
  - Replace `SUMMARY_PROMPT` with a JSON-returning meeting prompt
  - Replace `summarize_with_claude()` return type: now returns a `dict | None`
  - Replace `save_docx()` with a rewritten version that uses the dict and builds proper Word formatting

---

### Task 1: Rewrite the Claude prompt to return structured JSON

**Files:**
- Modify: `plaud_summarizer.py` — `SUMMARY_PROMPT` constant (lines ~38–49)

- [ ] **Step 1: Replace `SUMMARY_PROMPT`**

Replace the existing constant with:

```python
SUMMARY_PROMPT = """You are an expert meeting notes assistant. Analyze this work meeting transcript and return a JSON object with EXACTLY this structure (no markdown, no code fences, raw JSON only):

{{
  "attendees": ["Name or role if identifiable, else []"],
  "tldr": "2-3 sentence executive summary of the meeting",
  "decisions": [
    "Decision made (be specific)"
  ],
  "action_items": [
    {{"task": "What needs to be done", "owner": "Person responsible or Unknown", "deadline": "Date/timeframe or None"}}
  ],
  "topics": [
    {{
      "title": "Topic or agenda item",
      "points": ["Key point", "Key point"]
    }}
  ],
  "open_questions": [
    "Unresolved question or follow-up item"
  ]
}}

Rules:
- Be thorough. Capture every decision, action item, and key point.
- For action items: infer ownership from context (e.g. "John will...") even if not explicitly stated.
- If a field has no content, use an empty list [] or empty string "".
- topics should be organized by subject discussed, not by speaker.
- Return ONLY the JSON object. No explanation, no markdown.

Transcript:
{transcript}"""
```

- [ ] **Step 2: Verify prompt looks correct in file**

Read lines 38–70 of `plaud_summarizer.py` to confirm the new prompt is in place.

---

### Task 2: Update `summarize_with_claude()` to parse and return JSON

**Files:**
- Modify: `plaud_summarizer.py` — `summarize_with_claude()` function (lines ~82–115)

- [ ] **Step 1: Rewrite the function**

Replace the entire `summarize_with_claude` function with:

```python
def summarize_with_claude(transcript: str) -> dict | None:
    """Summarize transcript using Claude API. Returns structured dict or None."""
    if not ANTHROPIC_API_KEY:
        print("   No ANTHROPIC_API_KEY found. Skipping AI summary.")
        return None

    key = ANTHROPIC_API_KEY.strip()
    try:
        key.encode('ascii')
    except UnicodeEncodeError:
        print("   ANTHROPIC_API_KEY contains non-ASCII characters. Re-set it.")
        return None

    try:
        import anthropic
    except ImportError:
        print("Anthropic SDK not installed. Run: pip install anthropic")
        return None

    client = anthropic.Anthropic(api_key=key)
    print(f"    Summarizing with Claude ({CLAUDE_MODEL})...")

    message = client.messages.create(
        model=CLAUDE_MODEL,
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": SUMMARY_PROMPT.format(transcript=transcript)
        }]
    )

    raw = message.content[0].text.strip()

    try:
        import json
        data = json.loads(raw)
        print("    Summary generated")
        return data
    except json.JSONDecodeError:
        print("    Warning: Claude did not return valid JSON. Saving raw text as tldr.")
        return {"attendees": [], "tldr": raw, "decisions": [], "action_items": [], "topics": [], "open_questions": []}
```

- [ ] **Step 2: Verify the function signature and return type are correct**

Read the updated function in the file.

---

### Task 3: Rewrite `save_docx()` to build a professional Word document from JSON

**Files:**
- Modify: `plaud_summarizer.py` — `save_docx()` function (lines ~118–213)

- [ ] **Step 1: Rewrite `save_docx()`**

Replace the entire function with:

```python
def save_docx(audio_path: Path, summary: dict | None, output_dir: Path):
    """Save a formatted Word document from structured meeting summary."""
    try:
        from docx import Document
        from docx.shared import Pt, RGBColor, Inches
        from docx.enum.text import WD_ALIGN_PARAGRAPH
        from docx.oxml.ns import qn
        from docx.oxml import OxmlElement
    except ImportError:
        print("python-docx not installed. Run: pip install python-docx")
        return None

    doc = Document()

    # Page margins
    for section in doc.sections:
        section.top_margin = Inches(1)
        section.bottom_margin = Inches(1)
        section.left_margin = Inches(1.25)
        section.right_margin = Inches(1.25)

    # ── Title ─────────────────────────────────────────────────────────────────
    mtime = audio_path.stat().st_mtime
    recording_date = datetime.datetime.fromtimestamp(mtime).strftime("%B %d, %Y")
    stem = audio_path.stem.replace("_", " ").replace("-", " ").title()

    title = doc.add_heading(stem or "Meeting Notes", 0)
    title.alignment = WD_ALIGN_PARAGRAPH.LEFT
    for run in title.runs:
        run.font.color.rgb = RGBColor(0x1A, 0x1A, 0x2E)

    meta = doc.add_paragraph()
    meta.add_run(f"{recording_date}").font.size = Pt(10)
    meta.runs[0].font.color.rgb = RGBColor(0x88, 0x88, 0x88)
    doc.add_paragraph()

    if not summary:
        doc.add_paragraph("No AI summary generated (set ANTHROPIC_API_KEY to enable).")
    else:
        # ── Attendees ─────────────────────────────────────────────────────────
        attendees = summary.get("attendees", [])
        if attendees:
            doc.add_heading("Attendees", 1)
            doc.add_paragraph(", ".join(attendees))
            doc.add_paragraph()

        # ── TL;DR ─────────────────────────────────────────────────────────────
        tldr = summary.get("tldr", "")
        if tldr:
            doc.add_heading("Summary", 1)
            doc.add_paragraph(tldr)
            doc.add_paragraph()

        # ── Key Decisions ─────────────────────────────────────────────────────
        decisions = summary.get("decisions", [])
        if decisions:
            doc.add_heading("Key Decisions", 1)
            for d in decisions:
                p = doc.add_paragraph(style="List Bullet")
                p.add_run(d)
            doc.add_paragraph()

        # ── Action Items (table) ───────────────────────────────────────────────
        action_items = summary.get("action_items", [])
        if action_items:
            doc.add_heading("Action Items", 1)
            table = doc.add_table(rows=1, cols=3)
            table.style = "Table Grid"
            hdr = table.rows[0].cells
            for i, label in enumerate(["Task", "Owner", "Deadline"]):
                hdr[i].text = label
                for para in hdr[i].paragraphs:
                    for run in para.runs:
                        run.bold = True
                        run.font.size = Pt(10)

            # Set column widths
            col_widths = [Inches(3.2), Inches(1.5), Inches(1.3)]
            for i, width in enumerate(col_widths):
                for cell in table.columns[i].cells:
                    cell.width = width

            for item in action_items:
                row = table.add_row().cells
                row[0].text = item.get("task", "")
                row[1].text = item.get("owner", "Unknown")
                row[2].text = item.get("deadline", "") or "—"
                for cell in row:
                    for para in cell.paragraphs:
                        for run in para.runs:
                            run.font.size = Pt(10)
            doc.add_paragraph()

        # ── Discussion Topics ──────────────────────────────────────────────────
        topics = summary.get("topics", [])
        if topics:
            doc.add_heading("Discussion Topics", 1)
            for topic in topics:
                doc.add_heading(topic.get("title", ""), 2)
                for point in topic.get("points", []):
                    p = doc.add_paragraph(style="List Bullet")
                    p.add_run(point)
            doc.add_paragraph()

        # ── Open Questions ─────────────────────────────────────────────────────
        open_questions = summary.get("open_questions", [])
        if open_questions:
            doc.add_heading("Open Questions", 1)
            for q in open_questions:
                p = doc.add_paragraph(style="List Bullet")
                p.add_run(q)
            doc.add_paragraph()

    # ── Date Footer ───────────────────────────────────────────────────────────
    footer_text = doc.add_paragraph(recording_date)
    footer_text.alignment = WD_ALIGN_PARAGRAPH.CENTER
    for run in footer_text.runs:
        run.font.size = Pt(9)
        run.font.color.rgb = RGBColor(0xAA, 0xAA, 0xAA)

    # ── Save ──────────────────────────────────────────────────────────────────
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    out_name = f"{audio_path.stem}_{timestamp}.docx"
    out_path = output_dir / out_name
    doc.save(out_path)
    print(f"    Saved  {out_path}")
    return out_path
```

- [ ] **Step 2: Verify the function looks correct in file**

Read the updated `save_docx` function.

---

### Task 4: Update `process_file()` to match new signatures

**Files:**
- Modify: `plaud_summarizer.py` — `process_file()` function (lines ~216–228)

- [ ] **Step 1: Update the call to `save_docx`**

The old signature was `save_docx(audio_path, transcript, summary, output_dir)`.
The new one is `save_docx(audio_path, summary, output_dir)` — transcript is removed.

Replace `process_file()` with:

```python
def process_file(audio_path: Path, model, output_dir: Path):
    """Full pipeline: audio → transcript → summary → docx."""
    print(f"\n{'='*55}")
    print(f"  Processing: {audio_path.name}")
    print(f"{'='*55}")

    transcript = transcribe_audio(audio_path, model)
    summary = summarize_with_claude(transcript)
    out_path = save_docx(audio_path, summary, output_dir)

    if out_path:
        print(f"\n  Done!  {out_path}\n")
    return out_path
```

- [ ] **Step 2: Verify `process_file` in file**

Read the updated function.

---

### Task 5: Smoke test end-to-end

- [ ] **Step 1: Run a quick syntax check**

```bash
cd '/Users/hamadgul/Projects/Plaude Summarizer' && \
  .venv/bin/python3 -c "import plaud_summarizer; print('Import OK')"
```

Expected: `Import OK`

- [ ] **Step 2: Optionally test with a real audio file**

```bash
cd '/Users/hamadgul/Projects/Plaude Summarizer' && \
  .venv/bin/python3 plaud_summarizer.py path/to/test.mp3
```

Check that the output `.docx` in `output/` has:
- Meeting title heading
- Date line
- Summary, Key Decisions, Action Items table, Discussion Topics sections
- No "Full Transcript" section
- No "Generated by Plaude Summarizer" footer text

---
