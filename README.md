# SYRA

A voice assistant for your Mac: you talk, it talks back, and it opens apps, runs searches and reads out the weather for you.

![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB)
![Platform](https://img.shields.io/badge/platform-macOS-lightgrey)
![Mistral](https://img.shields.io/badge/mistralai-%E2%89%A51.0.0-FA520F)

## What it is

A single-process Python program that loops on your microphone. Each thing you say is transcribed, classified into an intent, and either executed on the machine or answered by Mistral. Replies are spoken with Google Text-to-Speech and played through `afplay`.

Everything on-machine goes through `open -a` and `osascript`, so this is macOS-only by design.

## What it does

- Listens continuously, transcribes with Google's speech API, and speaks the reply out loud.
- Opens and closes applications by name — about fifty aliases are mapped across the two `app_mappings` tables in `Assistance_SYRA_Final.py`, and anything it can't find locally falls back to the web version in Safari.
- Answers weather questions with real conditions: it resolves the place to coordinates, then reads current temperature, humidity, feels-like and rain from Open-Meteo.
- Sends video requests to a YouTube search and general lookups to a Google search, opening Safari and bringing it to the front.
- Holds a normal conversation through Mistral, with the last six messages kept as context.
- Detects Hindi speech and translates it to English before deciding what you meant.
- Writes every exchange to `conversations.txt`, and hangs up by itself after three silent prompts.

## How it works

- `Assistance_SYRA_Final.py` — the mic loop and every action: `open_application`, `close_application`, `search_in_safari`, `search_videos_in_youtube`, `get_location_coordinates`, and `execute_system_command`, which dispatches on the detected intent.
- `ai_handler.py` — `EdithAIHandler.detect_system_command` is the router; `get_ai_response` is the Mistral call plus the rolling history. `OptimizedSyraHandler` (at the bottom of the main file) subclasses it to answer greetings from a lookup table without a network round trip.
- `mistral_config.py` — client, model and the persona system prompt.
- `translation_handler.py` — talks to `translate.googleapis.com` directly, because the `googletrans` package conflicts with everything else in the tree. `googletrans` is not imported anywhere, so it and the `httpx==0.13.3` / `httpcore==0.9.1` / `h11==0.9.0` pins it needed have been removed from `requirements.txt`, along with `selenium`, `nltk` and `scipy`, which are also unimported. The file now resolves against `mistralai>=1.0.0`.

The genuinely fiddly part is the order of intent detection, not any single feature. "What's the weather in Melbourne" is simultaneously a question, a search and a place lookup, so weather keywords are tested before the who/what/when patterns; "open" is ignored whenever the sentence also contains video words, so "open a video of X" doesn't launch an app called "a video". The router is a priority ladder, and the comments in `detect_system_command` record which orderings broke.

Geocoding is also done by asking Mistral for a lat/long rather than calling a geocoder, with a hardcoded table of Australian cities as the fallback when that call times out.

## Run it locally

macOS, Python 3.8 or newer, a microphone and a Mistral API key.

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export MISTRAL_API_KEY='your-key'
python3 test_mistral_setup.py        # checks the key and the translator
python3 Assistance_SYRA_Final.py
```

`MISTRAL_API_KEY` is the only environment variable the program reads. It must be exported in the shell — nothing calls `load_dotenv()`, so a `.env` file is not picked up. `python-dotenv` used to be listed as a dependency for that reason; it has been dropped rather than wired in, because exporting the key is what every entry point already expects.

The first run will ask for microphone access. If it doesn't, grant it to your terminal under System Settings → Privacy & Security → Microphone. `setup.py` walks through the same steps interactively and writes a `launch_syra.sh`.

## What it doesn't do yet

- **No wake word.** It starts listening at launch and keeps going until you say goodbye or stay quiet three times. `clap.py` is a clap detector that would fix this, but nothing imports it, and as written it starts its own loop at import time.
- **No licence.** There is no LICENSE file in this repository, so default copyright applies and nobody else has permission to reuse the code.
- **macOS only.** `osascript`, `open -a` and `afplay` have no fallback on Linux or Windows.
- **Nothing is tested automatically.** `test_mistral_setup.py` is a connectivity check that needs a live key and a network; the intent router, which is where the logic lives, has no tests at all.
