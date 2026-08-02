# Tadabbur

Tadabbur is an app that helps you understand and reflect on the Quran through conversation. Instead of digging through multiple tafsir books, translations, and recitation apps, you just ask a question in plain language and the app's AI agent finds the answer for you — grounded in real Quranic sources, not made up.

> ⚠️ **Heads up: this project is still under active development.**
> The agent can make mistakes, misread a question, or give an answer that isn't fully accurate. We're still adding guardrails and reliability checks around it, so please don't treat its answers as a final religious authority yet. Always cross-check anything important with a qualified scholar or a trusted tafsir. Think of this as an early, evolving tool — not a finished product.

## What can you actually do with it?

- **Ask about any verse** — "what does surah al-baqarah ayah 255 mean?" and get the ayah, translation, and context.
- **Get tafsir explanations** — the agent pulls from Ibn Kathir's exegesis to explain what a verse means and why.
- **Ask why a verse was revealed** — Asbab al-Nuzul (the historical background/reason behind a revelation) is searchable too.
- **Listen to recitation** — ask to hear a verse or surah recited, pick a reciter, and it opens an audio player.
- **See a verse written out** — ask to "read" or "recite" a verse and it shows you a high-resolution image of the verse in Arabic.
- **Hear Quranic stories** — ask for "the story of Musa" or "Noah's Ark" and the agent narrates it in a few illustrated segments, with AI-generated images for each part.
- **Ask about verses by theme** — "which ayahs talk about patience" or "verses about gratitude," searched semantically rather than by exact keyword.
- **Upload a file and ask about it** — attach a PDF or text file and ask questions about it in the same chat.
- **Talk instead of type** — there's voice input, and you can also have responses read back to you out loud.
- **Bookmark verses** and pick up where you left off — the app remembers the last verse you were reading.
- **Switch AI models** mid-conversation, keep chat history across sessions, and give feedback (like/dislike or report) on any response.

Kids get a different experience too — if a user sets their age to 12 or under during onboarding, the agent switches to simpler language and a gentler, more encouraging tone.

## The agent — how it actually works

The heart of Tadabbur is a single conversational agent (built with LangGraph/LangChain) that decides what to do with your question and picks the right tool for the job rather than just guessing an answer from memory. Here's what's under the hood:

**Language models.** All chat responses are generated through **Groq**, which hosts a few different models you can switch between: Llama 3.1 8B, Llama 3.3 70B, and OpenAI's open-weight GPT-OSS 120B/20B models (served on Groq's infrastructure, not OpenAI's own API).

**Its tools.** When you ask something, the agent doesn't just answer from memory — it reaches for a specific tool depending on what you asked for:
- A verse-lookup tool that pulls exact ayahs by surah, juz, ruku, manzil, or hizb.
- A tafsir search tool that does a vector (semantic) search over Ibn Kathir's tafsir, stored in Qdrant.
- An Asbab al-Nuzul search tool for the "why was this revealed" questions, also vector-searched in Qdrant.
- An audio tool that finds recitation files for a given reciter and verse range.
- A verse-image tool for rendering Arabic verse text as an image.
- A story tool that hands off to a separate story-writing agent when you ask for a narrative.

The embeddings behind the semantic search (tafsir and Asbab al-Nuzul) are generated through **Fireworks AI**, and stored/searched in a **Qdrant** vector database.

**Story mode.** Asking the main agent for a story ("tell me the story of Yunus") works for everyone — it calls the story tool, which writes a short multi-part narrative and generates an illustration for each part (faces of prophets are deliberately left out of the images, in line with Islamic tradition). There's also a dedicated "story mode" chat experience in the UI, but right now that full mode is limited to a small group of internal testers while we finish testing it — it's not yet open to everyone.

**Guardrails already in place.** The agent is instructed to only answer from what its tools return (not to invent verses or interpretations), to stay on Quranic topics, to cap how many verses it returns at once, and it automatically retries a tool call if something fails instead of just erroring out. That said, this is exactly the area we're still hardening — see the disclaimer above.

## Tech stack

- **Frontend:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS. Talks to the backend over WebSockets for chat, and REST for everything else (auth, bookmarks, profile).
- **Backend:** FastAPI (Python), LangGraph/LangChain for the agent, asyncpg for Postgres.
- **Database:** PostgreSQL via Supabase.
- **Vector search:** Qdrant.
- **AI services:** Groq (chat models), Fireworks AI (embeddings + speech-to-text), Murf AI (text-to-speech), Hugging Face (story illustrations).

## Getting started

### Backend

```bash
cd backend
python -m venv .venv
.venv\Scripts\activate        # on Windows
pip install -r requirements.txt
uvicorn main:app --reload     # runs at http://localhost:8000
```

### Frontend

```bash
cd frontend
npm install
npm run dev                   # runs at http://localhost:3000
```

You'll need both running at the same time, plus the environment variables below, for the chat to actually work end to end.

## Environment variables

### Frontend — `frontend/.env`

| Variable | What it's for |
|---|---|
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Powers the "Sign in with Google" button |
| `NEXT_PUBLIC_WEBSOCKET_URL` | Where the frontend connects for live chat (e.g. `ws://localhost:8000`) |
| `NEXT_PUBLIC_BACKEND_URL` | Base URL of the backend REST API (e.g. `http://localhost:8000`) |

### Backend — `backend/.env`

| Variable | What it's for |
|---|---|
| `DATABASE_URL` | Postgres connection string (Supabase) |
| `JWT_SECRET_KEY` | Signs and verifies login tokens |
| `SUPABASE_URL` / `SUPABASE_KEY` | Supabase project credentials — used for the database and for storing images |
| `SUPABASE_STORAGE_BUCKET` | Bucket name for user profile pictures (defaults to `profile-images`) |
| `GENERATED_IMAGES_BUCKET` | Bucket where AI-generated story illustrations are stored |
| `QDRANT_API_KEY` / `QDRANT_URL_ENDPOINT` | Qdrant Cloud credentials — the vector database behind tafsir and Asbab al-Nuzul search |
| `GROQ_AI_API_KEY` | Runs the chat models (Llama, GPT-OSS) that power the agent |
| `FIREWORKS_AI_API_KEY` | Powers embeddings for semantic search, and voice-input transcription |
| `MURF_AI_API_KEY` | Generates the text-to-speech audio when a response is read aloud |
| `HUGGING_FACE_API` | Generates the illustrations used in Quranic stories |
| `WEB_GOOGLE_CLIENT_ID` / `APP_GOOGLE_CLIENT_ID` | Verify Google sign-in tokens from the web app and mobile app |
| `FRONTEND_URL` | Your frontend's URL — used to configure CORS |
| `TADABBUR_PROJECT_URL` / `TADABBUR_API_KEY` | Credentials for a secondary internal service client |

Optional — only needed if you want password-reset emails to actually send (otherwise the reset code just prints to your console during local dev):

| Variable | What it's for |
|---|---|
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` / `FROM_EMAIL` | SMTP credentials for sending password-reset emails |

`APP_ENV` is also read (defaults to `production`) and just shows up on the health-check endpoint — you can leave it out for local dev.

## Git workflow

- `main` is protected — changes go through a PR with at least 1 approval, and are squash-merged.
- Branch names: `feature/xxx` for new features, `fix/xxx` for bug fixes.
- Never force-push to `main`.
