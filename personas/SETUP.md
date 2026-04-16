# Persona / supergroup setup

The agent now supports multiple personas (sub-agents), each with its own
instructions, allowed skills, conversation history, and Telegram topic. As
long as `TELEGRAM_PERSONA_CHAT_ID` is unset in `.env`, everything keeps
working in the original DM mode and routes to the `general` persona.

To turn on multi-topic routing you do this once.

## 1. Create the supergroup

In the Telegram client:

1. Create a new group (any name — only you'll see it).
2. **Convert it to a supergroup** by upgrading any setting (e.g. set a
   public link, or change the privacy from "public to all" to "private").
   You can revert that setting afterwards. The bot needs a supergroup, not
   a regular group.
3. Add your bot to the group.
4. Promote the bot to admin. Tick **"Manage topics"** and **"Send messages"**
   at minimum.
5. In the group settings, enable **Topics**.

## 2. Create the topics

Make one topic per persona you want to use. The bot will respond in the
topic the message was sent in. The default `general` topic always exists.

For the seed personas:

- `🧬 Health`
- `🇭🇺 Hungarian`

You can add more later as you create personas.

## 3. Discover the chat id and thread ids

In each topic, send `/whereami` to the bot. It replies with:

```
📍 chat_id: -100xxxxxxxxxx
thread_id: 5
```

The `chat_id` is the same in every topic (it's the supergroup id, always
starts with `-100`). The `thread_id` is unique per topic.

In the supergroup's **General** topic, `thread_id` will be `(none)` —
that's expected. Telegram represents the General topic as a null
`message_thread_id`. The default persona handles all messages with no
thread id, so the supergroup's general chat will also route to the
`general` persona.

## 4. Wire `.env`

Add the values you discovered:

```env
# Persona routing
TELEGRAM_PERSONA_CHAT_ID=-100xxxxxxxxxx
TELEGRAM_TOPIC_GENERAL=
TELEGRAM_TOPIC_HEALTH=3
TELEGRAM_TOPIC_HUNGARIAN=5
```

`TELEGRAM_TOPIC_GENERAL` can stay empty — the supergroup's general chat
has no thread id, and the `general` persona is also the fallback for any
message without a thread id, so it picks it up automatically.

## 5. Restart workers

```bash
docker compose restart horizon scheduler telegram
```

That's it. The next time you message in a topic, the bot will respond in
that topic, with that persona's instructions and tools. Scheduled outputs
(e.g. the daily 🩺 health summary) will land in the corresponding topic
because `HealthInterpreter` and `HealthDiary` declare
`personaSlug() = 'health'`.

## How it actually works under the hood

- `config/personas.php` declares the personas. Each persona has a memory
  dir (where its instructions live), an `allowed_skills` list (or `['*']`),
  and a `thread_id` (null until you wire it in `.env`).
- Incoming messages: `TelegramPoll` reads `message.message_thread_id` →
  `HandleIncomingMessage` calls `PersonaRegistry::findByThreadId()` →
  `PersonalAgentFactory::forPersonaAndTelegramUser()` builds an agent
  with that persona's instructions + filtered tool list + persona-scoped
  conversation key (`tg:<userid>:<slug>`).
- Outgoing scheduled messages: heartbeat skills implementing
  `BelongsToPersona` declare their slug. `AgentNotifier::notifyForSkill()`
  resolves the persona and posts into the right topic. Skills that don't
  implement the interface fall back to the default persona's destination,
  which in DM mode is your private chat (so existing skills keep working
  identically until you opt them in).

## Adding a new persona

For now, by hand:

1. Create `storage/agent/personas/<slug>/persona.md` with the persona's
   instructions.
2. Add an entry under `'personas' =>` in `config/personas.php`:
   ```php
   'marketing' => [
       'name' => 'Marketing',
       'memory_dir' => storage_path('agent/personas/marketing'),
       'allowed_skills' => ['WebFetch', 'MemoryEditor'],
       'thread_id' => env('TELEGRAM_TOPIC_MARKETING'),
   ],
   ```
3. Create a topic in the supergroup, send `/whereami`, copy the thread id
   into `.env`.
4. `docker compose restart horizon scheduler telegram`.

A `CreatePersona` meta-skill that does steps 1–3 from a Telegram message
is on the roadmap (Phase 6) but not built yet.

## Conversation isolation

Conversations are stored per (persona, user) tuple in the
`agent_conversations` table. The general persona keeps its legacy key
(`tg:<id>`) so its existing history is preserved. Other personas use
`tg:<id>:<slug>`. `/new` only clears the conversation for the topic it's
sent in — your other personas' histories are untouched.
