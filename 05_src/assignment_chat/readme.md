# AstroBot 🚀 — Your AI Guide to the Universe

A conversational AI assistant built with Gradio and the OpenAI API that answers questions about space exploration, tracks the International Space Station in real time, and performs astronomical calculations.

---

## Overview

AstroBot is a chat interface with a distinct personality — an enthusiastic, wonder-filled space guide. It uses **three backend services** to answer user queries and maintains **memory** across the conversation with automatic trimming when the context window grows too large.

---

## Services

### Service 1: API Calls — Live Space Data (Open-Notify)

**File:** `services/api_service.py`

Uses the **[Open-Notify](http://open-notify.org/)** public API (no API key required) to fetch:
- **ISS real-time position** — current latitude and longitude of the International Space Station
- **Current astronauts in space** — names and spacecraft for everyone currently in orbit

The raw JSON responses are **transformed into natural-language text** before being returned. For example, a latitude/longitude pair becomes a descriptive sentence about hemisphere and cardinal direction. This fulfills the requirement that API output not be returned verbatim.

### Service 2: Semantic Query — Space Knowledge Base (ChromaDB)

**Files:** `services/semantic_service.py`, `build_embeddings.py`, `data/space_knowledge.json`

A semantic search service over a curated knowledge base of **20 space exploration documents** covering missions, telescopes, astrophysics, rockets, and planetary science. Topics include Apollo 11, Mars rovers, Hubble, JWST, black holes, exoplanets, Voyager 1, the ISS, and more.

**Technology:**
- **ChromaDB** with file persistence (`chroma_db/` directory)
- **OpenAI `text-embedding-3-small`** for embeddings
- Cosine similarity search, returning the top 3 most relevant documents

**Embedding Process:**
1. Embeddings are generated **once** by running `python build_embeddings.py` before first launch.
2. The script reads `data/space_knowledge.json`, calls the OpenAI Embeddings API, and persists the results to `chroma_db/` using ChromaDB's `PersistentClient`.
3. During app runtime, ChromaDB loads from disk — no re-embedding occurs on startup.
4. The dataset is 20 documents totalling ~20 KB, well within the 40 MB limit.

> **Important:** Run `python build_embeddings.py` before launching the app. The `chroma_db/` directory (not committed to git) is required for the semantic service.

### Service 3: Function Calling — Routing & Astronomical Calculations

**File:** `services/function_service.py`

Uses **OpenAI Function Calling** to let the LLM intelligently invoke the right tool for each query. Five tools are defined:

| Tool | Description |
|------|-------------|
| `get_iss_location` | Calls the Open-Notify ISS API |
| `get_current_astronauts` | Calls the Open-Notify astronauts API |
| `search_space_knowledge` | Queries the ChromaDB semantic store |
| `convert_astronomical_units` | Converts between km, AU, light-years, parsecs, light-minutes |
| `calculate_light_travel_time` | Given a distance in km, returns light-travel time in human-readable units |

**Why Function Calling?**  
Function calling provides a clean, reliable routing mechanism. Rather than using fragile keyword matching or manually deciding which service to invoke, the LLM itself determines which tool(s) to call based on the user's intent. This also allows **multi-tool responses** — for example, AstroBot might search the knowledge base *and* perform a unit conversion in a single turn.

---

## User Interface

Built with **Gradio Blocks** using a dark indigo/cyan theme.

**Personality:** AstroBot speaks with enthusiasm and wonder, uses vivid space metaphors, references famous scientists/astronauts, and makes complex topics approachable.

**Features:**
- Chat bubbles with rocket avatar
- Example questions for quick onboarding
- Submit on Enter or button click
- Autofocus on message input

---

## Memory Management

- Full conversation history is maintained within each session.
- A sliding window of the **last 10 turns** (20 messages) is sent to the LLM on each request, preventing context overflow.
- The system prompt is always prepended and excluded from the trim window.
- If history exceeds the window, the oldest messages are silently dropped — the user never sees an error.

This is the **trim-to-last-N** strategy recommended by LangGraph's memory management documentation.

---

## Guardrails

### System Prompt Protection
Regex pattern matching detects attempts to extract or modify the system prompt (e.g., "reveal your instructions", "ignore previous", "what is your system prompt"). These requests receive a canned refusal.

### Restricted Topics
The following topics are blocked at the input level before reaching the LLM:
- **Cats and dogs** — any mention of cat/dog breeds or species
- **Horoscopes and Zodiac Signs** — including all 12 signs and astrology terms
- **Taylor Swift** — artist name and fan community terms

Blocked requests receive a friendly redirect toward space topics.

---
