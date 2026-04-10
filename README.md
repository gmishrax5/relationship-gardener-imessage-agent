# Relationship Gardener – an iMessage‑native relationship agent

> Text `grow` and it picks one important person from your inner circle, shows your last touch, and drafts a natural 1‑line check‑in you can send today.

Most “personal CRM” tools live in heavy dashboards and CRMs you never open. Relationship Gardener lives where your real relationships already are: in an iMessage thread.

You tell it who matters to you (friends, mentors, collaborators). Once a day, you text `grow`. It chooses the person you have not checked in on for the longest, surfaces a tiny bit of context, and drafts a short, human check‑in line you can send right away.

All via iMessage. No web UI, no app to open.

---

## Why this exists

- It is easy to forget to ping mentors, old teammates, or friends when you are busy.
- Traditional CRMs are overkill and live outside your daily flow.
- iMessage is where you already talk to your closest circle.

Relationship Gardener is a tiny agent that helps you nurture relationships by making it effortless to send **one meaningful check‑in per day**.

---

## How it works (user experience)

All interaction happens in a single iMessage conversation with the agent’s Apple ID.

### 1. Onboarding – define your “garden”

You start the conversation:

- User: `setup`  
- Agent:  
  - “Let’s keep your important people warm. Reply with a list like:  
    `people: mom, ankit, mentor-alice, solana-frens`.”
- User:  
  - `people: mom, ankit, mentor-alice`
- Agent:  
  - “Got it. I’ll focus on these names. You can add or rename them anytime.”

For v1, the agent just stores **names / labels**. Later, these labels can be mapped to real contacts or be used only as your mental tags.

### 2. Daily `grow` check‑in

Once your people are set up, you use it like this:

- User: `grow`
- Agent:
  - Picks the person you have not “touched” for the longest (or never touched yet).
  - Surfaces a very short context snippet (for example, what you logged last time):
    - “Last time with `mentor-alice` you said: `Thanks again for reviewing my PR.`”
  - Drafts a 1‑line check‑in:
    - “Here’s a message you can send:  
      `Hey Alice, just shipped a new feature on my project – thought you’d appreciate it. How have you been lately?`”

You can copy‑paste this into your actual chat with that person, or just edit it and send.

### 3. Logging that you reached out

After a short delay:

- Agent: “Did you send something to `mentor-alice`? (yes/no)”
- If you reply `yes`, the agent records **today’s date** as the new `lastTouchedAt` for that person.
- If `no`, it leaves the previous date so the person will be picked again soon.

### 4. Light history overview

At any time:

- User: `garden`
- Agent: “You’ve checked in with:  
  - mom (5 days ago)  
  - ankit (2 days ago)  
  - mentor-alice (today)”

No fancy graphs. Just enough visibility to feel like you are maintaining your garden.

---

## Why this fits the Photon iMessage‑Kit challenge

**Prompt from the challenge:**

> Build an iMessage‑native agent with Photon using `imessage-kit`.  
> Some ideas: daily briefing, restaurant decider, accountability agent.  
> “The best submission will be the one we didn't think of. That one's yours.”

Relationship Gardener is **not** a daily briefing, restaurant picker, or generic habit tracker. It is a **relationship agent** focused on people, not tasks.

- **Personal utility:** If you care about friendships, mentors, or collab leads, this solves the real problem of “I haven’t talked to X in ages” by nudging you to send one concrete message each day.
- **Conversation‑native:** The entire experience is text inside iMessage. No web app, no external dashboard.
- **Explainable in one sentence:**  
  - “Text `grow` and it tells you who from your inner circle to check in with today and what to say.”

---

## High‑level architecture

### Components

- **macOS host**  
  - Runs the iMessage‑Kit server app.  
  - Signed in to the Apple ID for the agent’s iMessage identity.  
  - Exposes a local HTTP/WebSocket API that the Node/TypeScript process calls.[^imessagekit-server]

- **Node/TypeScript agent service**  
  - Uses `@photon-ai/advanced-imessage-kit` to talk to the macOS server.[^advanced-kit]  
  - Listens for new messages (`setup`, `grow`, `garden`, `yes`, `no`).  
  - Computes which person to suggest and what to say.  
  - Sends replies back via iMessage.

- **(Optional) Photon agent layer**  
  - Encapsulates the reasoning/prompting for drafting the 1‑line check‑in.  
  - Stores and retrieves user-specific memory (people list, last touches) alongside a small database.[^photon]

- **Database**  
  - Stores:
    - `users` (iMessage handle, createdAt)  
    - `people` (id, userId, label)  
    - `touches` (id, userId, personId, lastTouchedAt, lastSnippet)

### Data model (simple)

For v1, a minimal relational model works:

- `users`
  - `id` – internal UUID
  - `imessage_handle` – phone/email from iMessage
  - `created_at`

- `people`
  - `id`
  - `user_id`
  - `label` – e.g. `"mentor-alice"`
  - `created_at`

- `touches`
  - `id`
  - `user_id`
  - `person_id`
  - `last_touched_at` – date
  - `last_snippet` – short text like `"Thanks again for reviewing my PR"`

On each `grow`, the service:

1. Fetches all `people` for the user.
2. LEFT JOINs the latest `touches` per person.
3. Sorts by `last_touched_at` ascending (nulls first).
4. Chooses the first person as today’s suggestion.

---

## Tech stack

- **Runtime:** Node.js 18+  
- **Language:** TypeScript  
- **Messaging SDK:** `@photon-ai/advanced-imessage-kit` on top of the iMessage‑Kit macOS server.[^advanced-kit]  
- **Agent / reasoning (optional but intended):** Photon  
- **Database:** SQLite / Postgres / Supabase (anything simple with SQL)  
- **Hosting:** macOS machine (laptop or Mac mini) for the iMessage side

---

## Requirements

- macOS machine with:
  - iMessage signed into the Apple ID you want to use for the agent.
  - iMessage‑Kit server installed and running.[^imessagekit-server]
- Node.js 18 or later.
- Git + a database (or SQLite file) accessible from the Node process.

---

## Getting started

### 1. Clone the repo

```bash
git clone https://github.com/<your-username>/relationship-gardener-imessage-agent.git
cd relationship-gardener-imessage-agent
```

### 2. Install dependencies

```bash
npm install
# or
pnpm install
```

### 3. Set up iMessage‑Kit on macOS

1. Download and install the iMessage‑Kit macOS server app following the official docs.  
2. Start the app and configure:
   - API key
   - Server URL (default: `http://localhost:1234`)  
3. Make sure it can see your Messages database and is connected.[^imessagekit-server][^advanced-kit]

### 4. Configure environment

Create a `.env` file:

```env
IMESSAGEKIT_SERVER_URL=http://localhost:1234
IMESSAGEKIT_API_KEY=your-api-key-here
DATABASE_URL=postgres://user:pass@localhost:5432/relationship_gardener
# Optional: Photon / LLM API keys
PHOTON_API_KEY=...
OPENAI_API_KEY=...
```

### 5. Run database migrations

If you are using a migration tool:

```bash
npm run db:migrate
```

(Or create the `users`, `people`, and `touches` tables manually.)

### 6. Start the agent

```bash
npm run dev
# or
npm start
```

The agent service will connect to the iMessage‑Kit server and start listening for new messages.

### 7. Talk to your agent

From your iPhone:

1. Open iMessage and start a chat with the Apple ID that is signed into the Mac running iMessage‑Kit.
2. Send `setup` to initialize.
3. Send a `people: ...` list.
4. The next day (or immediately for testing), send `grow` and see the suggestion.

---

## Implementation notes

- **No direct auto‑DMs to others in v1**  
  - For privacy and simplicity, the agent **does not** send messages directly to your contacts in v1. It only drafts text for you to send manually.
- **Minimal memory**  
  - The agent only stores what you explicitly provide: labels and whether you said you reached out. It does not scrape your full iMessage history in v1.
- **Photon usage**  
  - You can start with deterministic templates for the suggested check‑ins.  
  - Then gradually introduce Photon to generate more personalized lines based on your logs.[^photon]

---

## Roadmap / possible extensions

- Map people labels to real iMessage handles (so the agent can deep link into those chats).
- Use Photon more heavily to:
  - Summarize last real messages.
  - Tailor check‑in tone (casual vs professional).
- Weekly digest: “This week you checked in with N people; here are the conversations.”
- Light tags (`mentor`, `family`, `friends`) and per‑tag frequencies.

---

## License

MIT (or your preferred license).

---

[^imessagekit-server]: iMessage‑Kit is a type‑safe macOS SDK/server that exposes the Messages database and sending APIs so Node/TypeScript apps can send and receive iMessages programmatically.[web:17][web:38][web:41]  
[^advanced-kit]: `@photon-ai/advanced-imessage-kit` is the TypeScript SDK for connecting to the iMessage‑Kit server, listening to incoming messages, and sending replies from Node.[web:35][web:36][web:37][web:40]  
[^photon]: Photon provides the agent framework and memory layer that can be used to enhance reasoning and personalization for this iMessage agent.[web:4][web:11]
