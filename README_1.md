# AI Meeting Intelligence System

Send a voice note to a Telegram bot. About a minute later you get proper meeting minutes back: a summary, the decisions made, who is doing what and by when, and the questions nobody answered. Everything also gets logged to Airtable so nothing is lost.

I built this for Nigerian business meetings, where people switch between English, Igbo, Yoruba, Pidgin and Hausa in the same sentence. Most meeting tools assume clean English. This one expects the mix.

## What it does

- Takes a meeting as **audio** (voice note, WhatsApp recording, audio file) or as a **pasted transcript**
- Transcribes audio with ElevenLabs Scribe
- Sends the transcript to GPT OSS 120B on Groq, which pulls out:
  - a short summary
  - decisions
  - action items with assignee, email, deadline and priority
  - open questions
  - unclear items (parts it could not understand confidently)
- Logs the meeting and every action item to Airtable
- Replies in Telegram with formatted minutes

## The rule I cared most about

The model is not allowed to make things up. If nobody said who owns a task, the assignee is `null`. If nobody gave a deadline, the deadline is `null`. If a part of the recording is too garbled to understand, it goes into `unclear_items` with the original fragment quoted, instead of the model guessing.

Wrong minutes are worse than no minutes, because people act on them.

## How it works

There are two workflows.

**1. Meeting Intelligence System** (the engine)

```
Webhook
  └─ Has Audio?
       ├─ yes: Download Audio → Scribe Transcribe → Normalize Transcript ─┐
       └─ no  ────────────────────────────────────────────────────────────┤
                                                                          ▼
                                                    Groq - Extract Intelligence
                                                                          ▼
                                                             Parse and Validate
                                                   ┌──────────────┼──────────────┐
                                          Respond to Webhook   Log Meeting   Split Action Items
                                                                                   ▼
                                                                           Log Action Items
```

It is a plain webhook, so anything can call it: a bot, a form, a script, another workflow.

**2. MIS - Telegram Front Door** (the bot)

```
Telegram Trigger → Switch
   ├─ Audio: Extract File ID → Get a file → Build Audio URL → Send Audio to MIS ─┐
   └─ Text:  Send Transcript to MIS ─────────────────────────────────────────────┤
                                                                                 ▼
                                                            Format Reply → Send Reply
```

## Example request

```json
POST /webhook/meeting-intel
{
  "audio_url": "https://example.com/meeting.mp3",
  "attendees": "Chidi <chidi@example.com>, Amaka <amaka@example.com>"
}
```

Or send `transcript` instead of `audio_url`.

## Example response

```json
{
  "meeting_id": "MTG-482913",
  "summary": "The team reviewed stock levels and agreed to restock fast moving items before the weekend.",
  "decisions": ["Restock paracetamol and ORS before Friday"],
  "action_items": [
    {
      "task": "Call the supplier to confirm delivery",
      "assignee": "Chidi",
      "assignee_email": "chidi@example.com",
      "deadline": "Friday",
      "priority": "high"
    }
  ],
  "open_questions": ["Should we switch suppliers?"],
  "unclear_items": []
}
```

## Real world test

I tested it on a real meeting recording in mixed Igbo and English. Scribe struggled. It skipped most of the Igbo and got stuck repeating a line. Groq still pulled out the correct summary, decisions and action items from what was left. That was the bet behind the whole design: the transcript does not need to be perfect if the extraction step is careful.

## Stack

- n8n (self hosted on Railway, PostgreSQL)
- Groq API, openai/gpt-oss-120b
- ElevenLabs Scribe (scribe_v2) for speech to text
- Airtable
- Telegram Bot API

## Setup

1. Import both JSON files from the `workflows` folder into n8n.
2. Create these credentials in n8n:
   - **Groq**: Header Auth, Name `Authorization`, Value `Bearer YOUR_GROQ_KEY`
   - **ElevenLabs**: Header Auth, Name `xi-api-key`, Value `YOUR_ELEVENLABS_KEY`
   - **Airtable**: Personal Access Token with `data.records:read`, `data.records:write` and `schema.bases:read`
   - **Telegram**: your bot token from BotFather
3. Create an Airtable base with two tables:
   - **Meetings**: Meeting ID, Date, Summary, Decision, Attendees, Open Questions, Unclear Items
   - **Actions Items**: Tasks, Asignee, Asignee Email, Deadline, Priority (High/Medium/Low), Status (Open/Done), Meeting ID
4. In the two Airtable nodes, replace `YOUR_AIRTABLE_BASE_ID` and the table IDs with yours. Keep Typecast on.
5. Add `TELEGRAM_BOT_TOKEN` as an environment variable on your n8n server, and set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` so the Build Audio URL node can read it.
6. In the Telegram workflow, replace `YOUR-N8N-INSTANCE` in both HTTP Request nodes with your n8n domain.
7. Publish both workflows.

## Things I learned building it

- Raw transcripts break a JSON body. Wrapping the text in `JSON.stringify()` inside the Groq request fixed it.
- The ElevenLabs header name has to be exactly `xi-api-key` or you get a 401.
- WhatsApp voice notes forwarded to a Telegram bot arrive as a document with type `video/mp4`, not as audio. The Switch checks for voice, audio and document so all three work.
- Models get retired. This ran on Llama 3.3 70B until Groq shut that model down in August 2026 and every run started failing with "resource not found". Swapping one line to `openai/gpt-oss-120b` fixed it, and the prompt worked the same without any other changes.
- Pin your `N8N_ENCRYPTION_KEY` on Railway. I lost every saved credential once when a redeploy generated a new key.

## What's next

- Better transcription for Igbo and other Nigerian languages
- Optional email follow ups to each person with their action items (held back until there is an opt in, so the endpoint can't be used to spam people)
- Plugging this in as the premium module of a small business automation suite for Lagos pharmacies and supermarkets

## Author

Jay (Onah Chibuike Joshua), AI Automation Specialist, Lagos

- X: [@jayonflow](https://x.com/jayonflow)
- GitHub: [chibs3529](https://github.com/chibs3529)
