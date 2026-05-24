# Database Patterns

## Setup (database/__init__.py)

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from .models import Base
from config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=False)
async_session = async_sessionmaker(engine, expire_on_commit=False)

async def create_tables():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_session() -> AsyncSession:
    async with async_session() as session:
        yield session
```

## Middleware (middlewares/db.py)

```python
from typing import Any, Awaitable, Callable
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject
from database import async_session

class DatabaseMiddleware(BaseMiddleware):
    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any]
    ) -> Any:
        async with async_session() as session:
            data["session"] = session
            return await handler(event, data)
```

## Register in main.py

```python
from middlewares.db import DatabaseMiddleware
from database import create_tables

async def main():
    await create_tables()
    # ...
    dp.update.middleware(DatabaseMiddleware())
```

## Handler with DB session

```python
from sqlalchemy.ext.asyncio import AsyncSession

@router.message(CommandStart())
async def start_handler(message: Message, session: AsyncSession):
    user = await get_or_create_user(
        session,
        tg_id=message.from_user.id,
        username=message.from_user.username,
        full_name=message.from_user.full_name
    )
```

## Common Queries

```python
# Get all user IDs (for broadcast)
async def get_all_user_ids(session: AsyncSession) -> list[int]:
    result = await session.execute(select(User.tg_id))
    return result.scalars().all()

# Count users
async def count_users(session: AsyncSession) -> int:
    result = await session.execute(select(func.count(User.id)))
    return result.scalar()

# Update user
async def update_user(session: AsyncSession, tg_id: int, **kwargs):
    await session.execute(
        update(User).where(User.tg_id == tg_id).values(**kwargs)
    )
    await session.commit()
```
