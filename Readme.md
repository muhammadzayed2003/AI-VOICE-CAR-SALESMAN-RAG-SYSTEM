# Zee Motors AI Voice Sales Agent

A bilingual (English/Urdu) AI voice sales agent for a car showroom, built with **Vapi**, **n8n**, and **Supabase (pgvector)**. The agent — named **Riley** — answers customer questions about inventory, pricing, and specs over a live phone call, grounding every answer in real showroom data via a RAG pipeline instead of guessing.

## What it does

A customer calls in and talks to Riley like they would a real salesperson. Riley listens, understands English or Urdu, and whenever the customer asks about a specific car, price, or spec, she queries the showroom's actual inventory data before answering — no hallucinated cars, no made-up prices. She keeps responses short (phone-call length) and steers the conversation toward booking a test drive.

## Architecture

The system is two n8n workflows plus a Vapi voice assistant:

### 1. Data Ingestion Pipeline
Takes raw inventory data and turns it into searchable vector embeddings.

`Form Trigger (name, showroom, inventory PDF)` → `Extract from File (PDF → text)` → `Chunking (splits by "Car #N:" pattern, extracts model + price metadata per chunk)` → `Google Gemini Embeddings` → `Supabase Vector Store (insert into pgvector)`

### 2. Data Retrieval Pipeline
Runs in real time during a call, triggered by Vapi's tool-call mechanism.

`Webhook (POST /car-voice-agent-v2)` → `Code node (parses Vapi's tool-call payload, extracts query + toolCallId)` → `Supabase Vector Store (similarity search via Gemini embeddings)` → `Code node (formats matched chunks into a result string)` → `Respond to Webhook (returns result in Vapi's expected tool-response format)`

### 3. Voice Assistant (Vapi)
- **Model:** GPT-4.1 (OpenAI)
- **Transcription:** Deepgram Nova-3, with auto-fallback enabled
- **Voice:** Vapi TTS (Naina)
- **Persona:** Car sales agent at Zee Motors, Bahria Town, Islamabad — bilingual, warm, honest, keeps replies to 2-3 sentences, always uses the `search_inventory` tool instead of guessing at inventory facts
- **Tool integration:** calls the n8n retrieval webhook mid-conversation and speaks the returned result back to the customer

## Tech Stack

- **n8n** — workflow orchestration (ingestion + retrieval pipelines)
- **Vapi** — voice AI platform (STT, LLM orchestration, TTS, telephony)
- **Supabase (pgvector)** — vector database for inventory embeddings
- **Google Gemini** — text embeddings
- **GPT-4.1** — conversational reasoning
- **Deepgram Nova-3** — speech-to-text

## How a call flows end-to-end

1. Customer calls the Vapi number and starts talking to Riley.
2. Customer asks something like "what SUVs do you have under 50 lakh?"
3. Vapi triggers the `search_inventory` tool, hitting the n8n retrieval webhook with the query.
4. n8n embeds the query, runs a similarity search against Supabase, and returns the matching inventory chunks.
5. Riley uses the returned data to answer — grounded, not guessed.
6. Conversation continues toward booking a test drive.

## Setup

1. Import both workflows into n8n (`AI-VOICE-SALESMAN-RAG-SYSTEM.json`).
2. Connect Supabase and Google Gemini credentials.
3. Run the ingestion form once per showroom to populate inventory data.
4. Import `vapi-assistant.json` into Vapi, point its `search_inventory` tool at your n8n retrieval webhook URL.
5. Test with a live or web call.

## Notes

This is a workflow-engineering project — the intelligence lives in how the pipelines are wired (chunking strategy, RAG retrieval, tool-call handling) rather than in custom application code.

## Author

Built by **Muhammad Zayed** ([ZeeLinks](https://github.com/)) — AI automation & n8n specialist.
