# aiogram3-claude-skill

A Claude Code skill that makes Claude write proper Telegram bots — not the usual garbage with hardcoded tokens, wrong aiogram versions, and emoji pasted directly into button text.

Drop this skill into your project and Claude will follow real patterns instead of making stuff up.

---

## The problem

Ask Claude to write a Telegram bot and you get:

- aiogram 2 syntax in an aiogram 3 project
- `BOT_TOKEN = "1234567890:ABC..."` hardcoded in the file
- Regular emoji in button labels (`text="⚙️ Settings"`) instead of Telegram Premium emoji
- Flat file with 400 lines, no handlers folder, no routers
- Sync database calls inside async handlers
- `print()` for error logging

This skill fixes all of that.

---

## What it enforces

**Project structure** — handlers split by domain, keyboards in their own folder, middlewares, database layer. Not everything in `bot.py`.

**aiogram 3 patterns** — `Router()` per file, `include_routers()` in main, `DefaultBotProperties`, correct FSM with `StatesGroup`, `callback.answer()` always called.

**Telegram Premium emoji** — `icon_custom_emoji_id` in inline and reply buttons, `<tg-emoji emoji-id="...">` tags in messages. Never plain emoji in button text.

**Async database** — SQLAlchemy 2.0 async + asyncpg/aiosqlite, session injected via middleware, typed models with `Mapped[]`.

**Config via pydantic-settings** — `.env` file, `ADMIN_IDS` as a list, never hardcoded credentials.

**Broadcast system** — iterates users with `asyncio.sleep(0.05)` rate limiting, catches `TelegramForbiddenError` and `TelegramBadRequest` per user.

**HTML parse mode** — always. No Markdown.

---

## Installation

Download `tg-bot-skill.skill` and install it in Claude Code:

```bash
claude skill install tg-bot-skill.skill
```

Or copy `SKILL.md` and the `references/` folder into your project root and Claude Code will pick it up automatically.

---

## What Claude produces with this skill

**Before** — keyboard with plain emoji:
```python
InlineKeyboardButton(text="⚙️ Settings", callback_data="settings")
```

**After** — proper Premium emoji:
```python
InlineKeyboardButton(
    text="Settings",
    callback_data="settings",
    icon_custom_emoji_id="5870982283724328568"
)
```

---

**Before** — handler file soup:
```python
# bot.py — 600 lines, everything in one file
@dp.message_handler(commands=["start"])  # aiogram 2 syntax
async def start(message):
    ...
```

**After** — proper structure:
```
bot/
├── main.py
├── config.py
├── handlers/
│   ├── start.py
│   └── admin.py
├── keyboards/
│   ├── inline.py
│   └── reply.py
├── middlewares/
│   └── db.py
└── database/
    ├── models.py
    └── requests.py
```

---

**Before** — admin check every time inline:
```python
if message.from_user.id == 123456789:  # hardcoded
```

**After**:
```python
def is_admin(user_id: int) -> bool:
    return user_id in settings.ADMIN_IDS
```

---

## What's inside

```
tg-bot-skill/
├── SKILL.md                     # Main skill — all patterns and rules
└── references/
    ├── premium_emoji.md         # Full table of Premium emoji IDs
    └── database.md              # SQLAlchemy async setup and query patterns
```

The `references/` files are loaded by Claude only when relevant — so context isn't wasted on database docs when you're just adding a keyboard.

---

## Stack

- Python 3.11+
- aiogram 3.x
- SQLAlchemy 2.0 async
- asyncpg / aiosqlite
- pydantic-settings

---

## Contributing

If you have patterns that belong here — better middleware, webhook setup, payment handlers, i18n — open a PR.
