# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Next.js 14 (App Router) chat app built for the Firebase codelab ["Build AI-powered web apps with Gemini API Firebase Extensions"](https://firebase.google.com/codelabs/gemini-api-extensions-web). It is not a typical app where Firestore is queried by the client app logic directly to call an LLM — instead, writes to a `messages` Firestore collection are picked up by an installed **Firebase Extension** (Gemini API / Vertex AI extension) running server-side, which fills in the `response` field asynchronously. The client only ever reads/writes Firestore and Storage directly via the Firebase Web SDK; there is no custom backend/API route in this repo.

## Commands

- `npm run dev` — start dev server (http://localhost:3000)
- `npm run build` — production build
- `npm run start` — run production build
- `npm run lint` — Next.js/ESLint lint (no test suite exists in this repo)

## Architecture

### Codelab scaffolding — read before editing chat logic

[src/app/page.tsx](src/app/page.tsx) contains `// TODO: N. Replace code next line with this:` comments. This file is a partially-completed step of the codelab: the prompt sent to Firestore is currently the raw user message (`prompt: userMsg`), and incoming docs are *not* run through `prepareMessage`. The commented-out lines show the "completed" version that wires in `preparePrompt` (context injection + conversation rules) and `prepareMessage` (extracts injected context and follow-up-prompt JSON from the LLM response). When asked to "complete" or "fix" the chat flow, check these TODOs first rather than re-deriving the wiring from scratch.

### Data flow

1. `sendMessage` in [page.tsx](src/app/page.tsx) writes a doc to Firestore collection `users/{uid}/messages` with a `prompt` field.
2. The Gemini Firebase Extension (configured outside this repo, in the Firebase console/`firebase.json` extensions) watches that collection, calls the Gemini API, and writes `response` + `status` back onto the same doc.
3. `onSnapshot` in `page.tsx` streams doc changes into local `messages` state; `status.state` (`PROCESSING` / `COMPLETED` / `ERROR`) drives the "Generating..."/error UI in [chat-container.tsx](src/components/chat-container.tsx).
4. [prepare-prompt.ts](src/lib/prepare-prompt.ts) builds the actual prompt text sent to the extension: the *first* message in a conversation gets the full context block from [context.ts](src/lib/context.ts) (the conference session catalog) plus formatting/guardrail rules, separated from the user's text by a `\n---\n` delimiter; subsequent messages only get a timestamp prefix (the extension keeps conversation history itself).
5. [message.ts](src/lib/message.ts)'s `prepareMessage` reverses step 4 for display: it splits `injectedContext` back out of `prompt`, and parses a trailing ```json code block out of `response` (validated with the `ResponseData` zod schema) to populate `followUpPrompts` — the suggested-reply buttons shown above the chat input.

### Auth

[firebase-user.tsx](src/lib/firebase-user.tsx) defines `FirebaseUserProvider`, a context provider wrapping the whole app in [layout.tsx](src/app/layout.tsx). It gates all children behind Google sign-in (`signInWithPopup` + `GoogleAuthProvider`): unauthenticated users see [signin-container.tsx](src/components/signin-container.tsx) instead of the app. Any component needing the current user reads `useContext(FirebaseUserContext)`.

### Gallery page

[gallery/page.tsx](src/app/gallery/page.tsx) is a second, independent feature: users upload images to Firebase Storage (`gallery/{uid}/...`) via [file-input-container.tsx](src/components/file-input-container.tsx) (drag/drop, 2MB limit, images only), and a corresponding Firestore doc in the `gallery` collection tracks moderation state (`published`, `flagUid`). Other users can "flag" an image, which the security rules only allow to set `published: false` + `flagUid`, never to delete the doc directly. Image URLs are resolved by reading the Storage path string in `image` and calling `getDownloadURL`.

### Firebase security rules

[firestore.rules](firestore.rules) and [storage.rules](storage.rules) are the actual access-control enforcement (not just docs) — `users/{uid}/messages` is private per-user; `gallery` is readable by any signed-in user but writable only by the owner, with flagging carved out as a narrow exception. Any change to gallery/messages data shapes in the client code must stay compatible with these rules' `hasOnly(...)` field allowlists.

### Firebase config

[src/lib/firebase.config.js](src/lib/firebase.config.js) hardcodes a Firebase project config object (apiKey, projectId, etc. — these are public client identifiers, not secrets) and calls `initializeApp`. Per the README, this is meant to be replaced with the reader's own Firebase project config when following the codelab.
