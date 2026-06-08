# Follow-Up Felix

AI-powered Gmail follow-up automation. Label a thread, snooze it, and Felix sends contextual follow-ups via Claude if the recipient doesn't reply — then stands down.

Built with Google Apps Script + Anthropic Claude API. No servers, no subscriptions. Runs entirely inside your Google account.

---

## How it works

Felix uses three Gmail labels to track state:

| Label | Applied by | Meaning |
|---|---|---|
| `-Snoozed-AI-FollowUp` | You | Watch this thread |
| `FollowedUp-1` | Felix | First follow-up sent |
| `FollowedUp-2` | Felix | Final follow-up sent — Felix is done |

A time-based trigger calls `main()` on a schedule. For each watched thread, Felix:

1. Finds your last outbound message
2. Checks if any external party replied after it — if yes, clears labels and exits
3. Confirms you're still the last sender
4. Sends a 2–4 sentence F1 via Claude (requires thread to be in inbox, i.e. snooze woke it)
5. Waits 96 hours (configurable), then sends a 1–2 sentence final F2
6. Cleans up all labels after F2

Claude receives the last N messages as context and generates a reply based on the prompt instructions in `config.js`. Felix extracts the recipient's first name from sign-offs, greetings, or display names, with a sanitiser that rejects false positives.

---

## File structure

```
follow-up-felix/
├── config.js       ← The only file most users need to edit
├── felix.js        ← Main loop and per-thread decision logic
├── gmail.js        ← All Gmail read/write (labels, send, context)
├── ai.js           ← Multi-provider AI adapter (Claude, OpenAI, DeepSeek, Gemini)
├── recipients.js   ← Address resolution and name extraction
├── utils.js        ← String helpers, sanitizers, quote stripper
└── debug.js        ← Developer tooling (no sends, no label changes)
```

Each file owns a clear domain. If you want to change how recipients are resolved, go to `recipients.js`. If you want to swap Claude for another LLM, go to `claude.js`. If you want to change the follow-up tone or timing, go to `config.js`.

---

## Setup

### Prerequisites

- A Google account with Gmail
- A Claude API key from [console.anthropic.com](https://console.anthropic.com)

### 1. Create Gmail labels

Create these three labels in Gmail (exact spelling matters):

```
-Snoozed-AI-FollowUp
FollowedUp-1
FollowedUp-2
```

The leading `-` on the first label sorts it to the top of your label list.

### 2. Create the Apps Script project

Go to [script.google.com](https://script.google.com) → **New project**.

Create one file per source file in this repo. Apps Script doesn't support imports — all files share a global scope, so load order doesn't matter as long as all files are present.

Under **Services**, add **Gmail API**. This is required for `Gmail.Users.Messages.send`, which properly threads replies using `In-Reply-To` headers.

### 3. Add your API key

**Project Settings → Script Properties → Add property:**

| Property key | Value |
|---|---|
| `CLAUDE_API_KEY` | Your Anthropic API key *(if using Claude)* |
| `OPENAI_API_KEY` | Your OpenAI API key *(if using OpenAI)* |
| `DEEPSEEK_API_KEY` | Your DeepSeek API key *(if using DeepSeek)* |
| `GEMINI_API_KEY` | Your Google Gemini API key *(if using Gemini)* |

Only add the key for the provider you're using. Never paste keys directly into source files.

### 4. Edit `config.js`

The only configuration file. At minimum, set:

```javascript
// Addresses you send from (other than your primary Gmail)
const INTERNAL_EMAILS_EXTRA = [
  // "alias@yourdomain.com",
];

// Your company domain(s) — senders here are treated as internal
const INTERNAL_DOMAINS = [
  // "yourdomain.com",
];
```

Everything else (timing, prompts, model, sign-off name) has sensible defaults you can leave or adjust.

### 5. Set a trigger

**Triggers → Add Trigger:**

- Function: `main`
- Event source: Time-driven
- Type: Hours timer
- Interval: Every 2–4 hours

### 6. Use it

On any outbound thread you want to watch:

1. Apply the `-Snoozed-AI-FollowUp` label
2. Snooze the thread in Gmail

When the snooze expires and Gmail returns the thread to your inbox, Felix will pick it up on the next trigger run.

---

## Configuration reference

Everything is in `config.js`. Key options:

| Variable | Default | Description |
|---|---|---|
| `HOURS_BETWEEN_FOLLOWUPS` | `96` | Hours to wait between F1 and F2 |
| `CONTEXT_MESSAGE_COUNT` | `5` | Messages sent to Claude as context |
| `AI_PROVIDER` | `"claude"` | Which AI to use: `claude`, `openai`, `deepseek`, `gemini` |
| `AI_MODEL` | `"claude-sonnet-4-5"` | Model string for your chosen provider |
| `AI_MAX_TOKENS` | `300` | Max tokens for generated reply |
| `FELIX_NAME` | `"Follow-Up Felix"` | Sign-off name in all emails |
| `PROMPT_F1` | *(see file)* | Instructions for first follow-up |
| `PROMPT_F2` | *(see file)* | Instructions for final follow-up |
| `FALLBACK_F1` / `FALLBACK_F2` | *(see file)* | Used when Claude API fails |

---

## Debugging

All debug functions are in `debug.js`. None of them send emails or modify labels.

**Find a thread to inspect:**

```javascript
// By subject snippet (saves the thread ID automatically)
setDebugBySubject("Re: Your proposal")

// Or directly
setDebugThreadId("19b4b310c8274edd")
```

**Full status dashboard (dry-run):**

```javascript
debugFelix()
```

Prints: labels, stage, last sender, reply detection result, resolved To/CC, and whether F1/F2 would fire — with blockers listed if not.

**Other debug functions:**

```javascript
debugListMessages()   // All messages oldest → newest
debugReplyWindow()    // Messages after your last send
debugNamePick()       // Name extraction trace for the resolved To address
logThreadsInWatchLabel()  // First 10 threads in the watch label
testAiProvider()      // Smoke-test your configured AI provider
```

**Enable verbose logging:**

In Script Properties, add `FELIX_DEBUG` = `true`. Remove to silence.

---

## Extending Felix

### Swap the AI provider

Change `AI_PROVIDER` and `AI_MODEL` in `config.js` and add the corresponding API key in Script Properties. Supported out of the box: `claude`, `openai`, `deepseek`, `gemini`.

To add a new provider, add a `_callYourProvider()` function in `ai.js` following the same pattern as the existing ones, then add a `case` for it in the `generateReply()` switch.

### Add a third follow-up stage

1. Add `STAGE_3: "FollowedUp-3"` to `LABEL` in `config.js`
2. Add `PROMPT_F3` and `FALLBACK_F3`
3. Add a Pool C to `main()` in `felix.js`: `GmailApp.search('label:FollowedUp-2 -label:FollowedUp-3')`
4. Extend `processThread()` to handle the new stage

### Log to Google Sheets

Add a `logToSheet(threadId, to, subject, stage)` function in `gmail.js` and call it after each successful send in `felix.js`. Use `SpreadsheetApp.openById(YOUR_SHEET_ID)` to target a specific sheet.

### Personalise prompts per label

Add custom Gmail labels like `-FollowUp-Sales` vs `-FollowUp-Support` and read the thread's labels in `processThread()` to select different prompt templates from `config.js`.

---

## Notes

- Felix only fires when **you** were the last sender. If a recipient replied and you haven't responded yet, Felix skips the thread.
- F1 requires the thread to be in your inbox — the Gmail snooze must have expired. F2 fires even if the thread was archived after F1.
- The 96-hour F2 gate uses a timestamp stored in `PropertiesService`. If that property is missing, Felix scans the thread for its own signature as a fallback.
- After `FollowedUp-2`, the thread is removed from all watched pools. Felix never sends a third follow-up unless you extend it.

---

## License

MIT

---

*Built by [Aleph Creative](https://alephcreative.com). Read the build writeup →*
