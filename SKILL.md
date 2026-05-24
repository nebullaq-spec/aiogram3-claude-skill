---
name: tg-bot
description: >
  Expert skill for building production-grade Telegram bots in Python with aiogram 3.
  Use this skill whenever the user wants to create, modify, or extend a Telegram bot —
  whether it's handlers, FSM states, inline keyboards, broadcast systems, admin panels,
  subscription checks, databases, or any bot-related code. Always use this skill when
  the user mentions "telegram bot", "aiogram", "tg bot", "бот", "телеграм бот",
  or asks to write handlers, keyboards, or bot commands. Never write generic Python —
  always follow the patterns in this skill.
---

# Telegram Bot Skill (aiogram 3 + Python)

You are an expert Telegram bot developer. Always write clean, production-ready code
following the patterns below. Never deviate from the structure unless explicitly asked.

## Stack

- **Framework**: aiogram 3.x (NOT aiogram 2)
- **Language**: Python 3.11+
- **Database**: SQLAlchemy 2.0 async + asyncpg (PostgreSQL) or aiosqlite (SQLite)
- **Config**: pydantic-settings with `.env`
- **Parse mode**: always `ParseMode.HTML` (never Markdown)

---

## Project Structure

```
bot/
├── main.py                  # Entry point, dp + bot setup
├── config.py                # Pydantic settings
├── .env                     # Secrets (never commit)
├── handlers/
│   ├── __init__.py
│   ├── start.py
│   ├── admin.py
│   └── ...
├── keyboards/
│   ├── __init__.py
│   ├── inline.py
│   └── reply.py
├── middlewares/
│   ├── __init__.py
│   └── db.py
├── database/
│   ├── __init__.py
│   ├── models.py
│   └── requests.py
├── states/
│   └── __init__.py
└── utils/
    └── __init__.py
```

---

## Config (config.py)

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    BOT_TOKEN: str
    ADMIN_IDS: list[int]
    DATABASE_URL: str = "sqlite+aiosqlite:///bot.db"

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## Main Entry Point (main.py)

```python
import asyncio
from aiogram import Bot, Dispatcher
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties
from config import settings
from handlers import start, admin

async def main():
    bot = Bot(
        token=settings.BOT_TOKEN,
        default=DefaultBotProperties(parse_mode=ParseMode.HTML)
    )
    dp = Dispatcher()
    dp.include_routers(start.router, admin.router)
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Premium Emoji — CRITICAL RULES

**NEVER use regular emoji in bot messages or buttons.** Always use premium emoji via `<tg-emoji>` tags.

### In messages (HTML parse_mode):
```python
await message.answer(
    '<b><tg-emoji emoji-id="5870982283724328568">⚙️</tg-emoji> Настройки</b>\n\n'
    'Выберите раздел:'
)
```

### Premium Emoji Reference
Read `references/premium_emoji.md` for the full list of emoji IDs.
Always pick the most semantically appropriate emoji from the list.

### In InlineKeyboardButton:
```python
# NEVER put emoji in text= field directly
# ALWAYS use icon_custom_emoji_id=
InlineKeyboardButton(
    text="Настройки",
    callback_data="settings",
    icon_custom_emoji_id="5870982283724328568"
)
```

### In ReplyKeyboardButton:
```python
{"text": "Настройки", "icon_custom_emoji_id": "5870982283724328568"}
```

---

## Keyboards

### Inline Keyboard (keyboards/inline.py)
```python
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton

def get_main_menu() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(
            text="Профиль",
            callback_data="profile",
            icon_custom_emoji_id="5870994129244131212"
        )],
        [InlineKeyboardButton(
            text="Настройки",
            callback_data="settings",
            icon_custom_emoji_id="5870982283724328568"
        )],
        [InlineKeyboardButton(
            text="Назад",
            callback_data="back",
            icon_custom_emoji_id="5893057118545646106"
        )],
    ])
```

### Reply Keyboard (keyboards/reply.py)
```python
from aiogram.types import ReplyKeyboardMarkup

def get_main_keyboard() -> ReplyKeyboardMarkup:
    keyboard = {
        "keyboard": [
            [{"text": "Профиль", "icon_custom_emoji_id": "5870994129244131212"}],
            [
                {"text": "Настройки", "icon_custom_emoji_id": "5870982283724328568"},
                {"text": "Помощь", "icon_custom_emoji_id": "6028435952299413210"}
            ],
        ],
        "resize_keyboard": True
    }
    return ReplyKeyboardMarkup(**keyboard)
```

---

## Handlers

### Basic Handler Pattern
```python
from aiogram import Router, F
from aiogram.types import Message, CallbackQuery
from aiogram.filters import CommandStart

router = Router()

@router.message(CommandStart())
async def start_handler(message: Message):
    await message.answer(
        f'<b><tg-emoji emoji-id="6041731551845159060">🎉</tg-emoji> Привет, {message.from_user.first_name}!</b>\n\n'
        'Выбери действие:',
        reply_markup=get_main_menu()
    )

@router.callback_query(F.data == "profile")
async def profile_callback(callback: CallbackQuery):
    await callback.message.edit_text(
        '<b><tg-emoji emoji-id="5870994129244131212">👤</tg-emoji> Профиль</b>',
        reply_markup=get_back_keyboard()
    )
    await callback.answer()
```

### Always call `callback.answer()` at the end of every callback handler.
### Use `edit_text` for inline message updates, `answer` for new messages.

---

## FSM (States)

```python
from aiogram.fsm.state import State, StatesGroup

class BroadcastStates(StatesGroup):
    waiting_for_message = State()

class UserStates(StatesGroup):
    waiting_for_name = State()
    waiting_for_phone = State()
```

### FSM Handler Pattern
```python
from aiogram.fsm.context import FSMContext

@router.message(BroadcastStates.waiting_for_message)
async def process_broadcast(message: Message, state: FSMContext):
    await state.update_data(broadcast_text=message.text)
    data = await state.get_data()
    await state.clear()
    # process...
```

---

## Admin Panel

```python
from config import settings

def is_admin(user_id: int) -> bool:
    return user_id in settings.ADMIN_IDS

@router.callback_query(F.data == "admin_broadcast")
async def broadcast_callback(callback: CallbackQuery, state: FSMContext):
    if not is_admin(callback.from_user.id):
        await callback.answer(
            '<tg-emoji emoji-id="5870657884844462243">❌</tg-emoji> Нет доступа',
            show_alert=True
        )
        return

    await callback.message.edit_text(
        '<b><tg-emoji emoji-id="6039422865189638057">📣</tg-emoji> Рассылка</b>\n\n'
        'Отправь сообщение для рассылки:',
        reply_markup=get_cancel_keyboard()
    )
    await state.set_state(BroadcastStates.waiting_for_message)
    await callback.answer()
```

---

## Broadcast System

```python
from aiogram import Bot
from aiogram.exceptions import TelegramForbiddenError, TelegramBadRequest
import asyncio

async def send_broadcast(bot: Bot, user_ids: list[int], text: str) -> dict:
    success, failed = 0, 0
    for user_id in user_ids:
        try:
            await bot.send_message(user_id, text)
            success += 1
        except (TelegramForbiddenError, TelegramBadRequest):
            failed += 1
        await asyncio.sleep(0.05)  # rate limit
    return {"success": success, "failed": failed}
```

---

## Channel Subscription Check

```python
from aiogram import Bot
from aiogram.types import ChatMemberStatus

CHANNEL_ID = "@your_channel"

async def check_subscription(bot: Bot, user_id: int) -> bool:
    try:
        member = await bot.get_chat_member(CHANNEL_ID, user_id)
        return member.status not in [
            ChatMemberStatus.LEFT,
            ChatMemberStatus.KICKED,
            ChatMemberStatus.BANNED
        ]
    except Exception:
        return False

def get_subscribe_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(
            text="Подписаться",
            url=f"https://t.me/your_channel",
            icon_custom_emoji_id="6039422865189638057"
        )],
        [InlineKeyboardButton(
            text="Проверить подписку",
            callback_data="check_subscribe",
            icon_custom_emoji_id="5870633910337015697"
        )],
    ])
```

---

## Database (SQLAlchemy async)

```python
# database/models.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import BigInteger, String, DateTime, func

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    tg_id: Mapped[int] = mapped_column(BigInteger, unique=True)
    username: Mapped[str | None] = mapped_column(String(64))
    full_name: Mapped[str] = mapped_column(String(128))
    created_at: Mapped[DateTime] = mapped_column(default=func.now())
```

```python
# database/requests.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from .models import User

async def get_or_create_user(session: AsyncSession, tg_id: int, **kwargs) -> User:
    result = await session.execute(select(User).where(User.tg_id == tg_id))
    user = result.scalar_one_or_none()
    if not user:
        user = User(tg_id=tg_id, **kwargs)
        session.add(user)
        await session.commit()
    return user
```

---

## Error Handling

- Always wrap bot API calls in try/except
- Log errors with `logging` module, not `print`
- Never expose internal errors to users — show friendly messages with emoji

```python
import logging
logger = logging.getLogger(__name__)

try:
    await bot.send_message(user_id, text)
except Exception as e:
    logger.error(f"Failed to send message to {user_id}: {e}")
```

---

## Security Rules

1. **Never hardcode** BOT_TOKEN, ADMIN_IDS, or DB credentials — always use `.env`
2. **Always validate** admin access before executing admin commands
3. **Rate limiting**: add `asyncio.sleep(0.05)` in loops over users
4. **Input validation**: sanitize user input before storing in DB

---

## Code Style Rules

1. All handlers in separate files by domain (start, admin, profile, etc.)
2. All keyboards in `keyboards/` directory
3. Router per file: `router = Router()` at top of each handler file
4. Always `include_routers()` in main.py
5. Type hints everywhere
6. Async everywhere — never use sync DB calls
7. HTML parse mode always — escape user input with `html.escape()`

---

## References

- Full premium emoji list with IDs: `references/premium_emoji.md`
- Database setup patterns: `references/database.md`
- Middleware patterns: `references/middlewares.md`
