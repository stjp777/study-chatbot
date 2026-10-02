# Study Chatbot — v1 Design

## Intent

A chatbot that helps a student study using a technique suited to their situation. Built for personal/small-group use (the team), deployed on Vercel so teammates can access it from a shared URL.

**Scope decision:** the original idea ("recommend techniques" + "generate materials" + "interactive tutor") is three subsystems. This spec covers a single combined slice: recommend *and run* one technique in one chat session. No persistence, no accounts, no multi-session tracking in v1.

**Explicitly deferred to later phases (do not lose track of these):**
- **Saving study progress across sessions** (requested by the user during design — flashcard decks, spaced-repetition scheduling, history should persist instead of being discarded on tab close).
- Accounts / multi-user support.
- Additional techniques beyond flashcards and Socratic questioning.
- Material upload beyond single-file text extraction (e.g. multi-file, OCR of scanned/image-only PDFs).

## Success Criteria

- A student can paste text and/or upload a .txt/.pdf/.pptx file of study material.
- The bot asks whether the material is new or review, and runs the matching technique without the student having to configure anything.
- Flashcards render as real interactive flip-card UI, not plain text.
- Socratic mode behaves conversationally — it guides rather than answers outright.
- Runs locally (`npm run dev`) and deploys to Vercel.

## Architecture

- **Framework:** Next.js (App Router), deployed on Vercel.
- **AI:** Vercel AI SDK, using the AI Gateway with a plain `"provider/model"` string (e.g. `anthropic/claude-sonnet-5`), not a provider-specific SDK — swapping models later is a config change.
- **Chat UI:** AI SDK's `useChat` hook. One API route, `/api/chat`, runs `streamText` with a system prompt instructing the model to:
  1. Always ask "is this new to you, or are you reviewing it?" before proceeding.
  2. On "new" → run Socratic questioning as plain streamed conversation (no special UI — it's inherently conversational).
  3. On "review" → call a `generateFlashcards` tool.
- **Flashcards as generative UI:** `generateFlashcards` returns a structured `{question, answer}[]` array (tool call with schema validation). Rendered client-side as real flip-card components below the chat, not as chat text. Student can ask for more/different cards, which re-invokes the tool.
- **File upload:** a file input in the chat UI. Uploaded file is parsed server-side with `officeparser` (one library, handles .txt/.pdf/.pptx uniformly) inside `/api/chat` before the model call — no separate upload endpoint for v1. Extracted text is merged with any pasted text into a single "material" context.
- **No backend state or database.** Conversation lives in React state for the lifetime of the browser tab. This is the one piece of this design phase 2 (saving progress) will replace.

## Data Flow

```
Browser (useChat, React state)
  ⇄ POST /api/chat (Next.js route)
      - if file attached: officeparser extracts text
      - merge pasted text + extracted text → "material"
      - streamText(system prompt + material + conversation) via AI SDK
  ⇄ Vercel AI Gateway
  ⇄ model
```

Tool calls (`generateFlashcards`) happen within the same `streamText` call via AI SDK tool-calling; results stream back to the client and are rendered by a dedicated flashcard component keyed off the tool result.

## Error Handling

- **Corrupt/unsupported file:** `officeparser` failure is caught; user gets a plain chat message ("couldn't read that file — try pasting the text instead") instead of a crash.
- **File too large:** rejected client-side before upload (e.g. 10MB limit), since model context has limits too.
- **Model/Gateway errors** (rate limit, timeout): surfaced via `useChat`'s built-in error state as a retryable message, not a silent hang.
- **No material provided:** handled conversationally — the system prompt has the bot ask again — no special app-level validation branch needed.

## Testing

Chat behavior itself is prompt-driven and best verified by hand; code around it gets unit tests:
- Unit test: file-parsing helper — given sample .txt/.pdf/.pptx fixtures, returns expected extracted text; given a bad/corrupt file, returns a handled error (not a throw that crashes the route).
- Unit test: `generateFlashcards` tool schema validation — malformed/partial model output doesn't crash rendering.
- Manual verification before calling any UI work done: run dev server, actually paste notes and upload a sample file of each supported type, walk through both the Socratic and flashcard branches in a real browser.

## Out of Scope for v1 (tracked for later)

- Persistent study progress / saved flashcard decks / spaced repetition scheduling.
- User accounts.
- Additional study techniques.
- Multi-file or advanced document parsing (OCR, scanned PDFs).
