# YouTube Transcript Summarizer

I watch a lot of YouTube for learning — lectures, conference talks, tutorials. The problem is a 45-minute video often has maybe 10 minutes of actual content worth keeping, and there's no fast way to know which 10 minutes those are before you've watched the whole thing. So I built this.

It's a Chrome extension that sits in your browser toolbar. You're on any YouTube video, you click it, and within a few seconds you get a real summary — not a list of timestamps, not an auto-generated description, but an actual abstractive summary that reads like someone watched the video and told you what happened.

The backend is a Flask API running locally, and the model is `facebook/bart-large-cnn` from Hugging Face — a transformer fine-tuned on CNN/DailyMail articles for abstractive summarization. I specifically didn't want to route everything through the OpenAI API because (a) cost adds up and (b) I wanted to understand what it takes to run inference locally and serve it through an API I control.

## The chunking problem

This was the hardest part, and it's not obvious until you hit it.

BART has a 1024 token input limit. YouTube transcripts for anything longer than a short video easily hit 8,000–15,000 tokens. The naive approach — truncate at 1024 — gives you a summary of the first few minutes and ignores everything else. Useless.

The solution I landed on is a two-pass pipeline:

1. Split the transcript into overlapping chunks of ~900 tokens (overlap helps preserve context at boundaries)
2. Summarize each chunk independently
3. Concatenate the chunk summaries into a single intermediate text
4. Run one final summarization pass on that intermediate text

The second pass is the key part — it stitches the chunk summaries into something coherent rather than just concatenating disconnected fragments. After adding this, the quality on long videos improved significantly. It's slower (15–25 seconds for an hour-long video) but the output is actually useful.

## How to run it

You'll need Python and pip, and a Chromium-based browser.

```bash
git clone https://github.com/radhapawar/Youtube_Transcript_Summarizer.git
cd Youtube_Transcript_Summarizer
pip install flask youtube-transcript-api transformers torch
python app.py
```

First run will download the BART model (~1.6 GB) and cache it. After that it loads in a few seconds.

Then load the extension:
- Go to `chrome://extensions/`
- Turn on Developer Mode (top right toggle)
- Click "Load unpacked" and point it at the repo folder
- The extension icon appears in your toolbar

Go to any YouTube video and click it.

## Stack

- **Chrome Extension** (Manifest V3) — JavaScript, HTML, CSS
- **Backend** — Python, Flask
- **Model** — `facebook/bart-large-cnn` via Hugging Face `transformers`
- **Transcript fetching** — `youtube-transcript-api`

## What I'd do differently

If I built this again I'd deploy the Flask backend to a cloud endpoint instead of requiring a local server — probably AWS Lambda with the model stored in S3. Running it locally means it only works when your machine is on, which is a real limitation. I'd also experiment with `google/pegasus-xsum` which tends to produce more compressed, newspaper-headline-style summaries that might actually be better for the quick-check use case this is designed for.

There's also an interesting unsolved problem around multilingual transcripts — BART is English-only, so for non-English videos you'd need to translate first, which adds latency and another moving part. Solvable, just haven't gotten there yet.
