import asyncio
import aiosqlite
import os
from aiogram import Bot, Dispatcher, types
from aiogram.filters import CommandStart
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# Використовуємо змінну оточення для безпеки
TOKEN = os.getenv("8512057548:AAGXTagRc2J4PVnlV_sJ_iqr606qH7Duc08")

bot = Bot(token=TOKEN)
dp = Dispatcher()

DB_NAME = "bot.db"

DAILY_NORM = 10        # щоденна норма запрошених
REWARD = 80            # грн за одного запрошеного
MIN_BALANCE = 10000    # мінімальна сума для виводу
MIN_DAYS = 10          # мінімальна кількість робочих днів


# 📋 Меню
menu = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="📊 Статистика")],
        [KeyboardButton(text="🔗 Моє посилання")],
        [KeyboardButton(text="💸 Вивести кошти")]
    ],
    resize_keyboard=True
)


# 🗄️ Ініціалізація бази
async def init_db():
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            invited INTEGER DEFAULT 0,
            balance INTEGER DEFAULT 0,
            day INTEGER DEFAULT 1
        )
        """)

        await db.execute("""
        CREATE TABLE IF NOT EXISTS invites (
            user_id INTEGER,
            invited_today INTEGER DEFAULT 0
        )
        """)

        await db.execute("""
        CREATE TABLE IF NOT EXISTS referrals (
            user_id INTEGER UNIQUE,
            referrer_id INTEGER
        )
        """)

        await db.commit()


# 👤 Реєстрація нового користувача
async def register(user_id):
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        await db.execute("INSERT OR IGNORE INTO invites (user_id) VALUES (?)", (user_id,))
        await db.commit()


# 🚀 Старт бота
@dp.message(CommandStart())
async def start(message: types.Message):
    user_id = message.from_user.id
    args = message.text.split()

    await register(user_id)

    # Лінк із рефералом
    if len(args) > 1:
        ref_id = int(args[1])

        if ref_id != user_id:
            async with aiosqlite.connect(DB_NAME) as db:
                cur = await db.execute("SELECT * FROM referrals WHERE user_id=?", (user_id,))
                if not await cur.fetchone():

                    await db.execute(
                        "INSERT INTO referrals (user_id, referrer_id) VALUES (?, ?)",
                        (user_id, ref_id)
                    )

                    await db.execute(
                        "UPDATE users SET invited = invited + 1, balance = balance + ? WHERE user_id=?",
                        (REWARD, ref_id)
                    )

                    await db.execute(
                        "UPDATE invites SET invited_today = invited_today + 1 WHERE user_id=?",
                        (ref_id,)
                    )

                    await db.commit()

                    cur = await db.execute("SELECT invited, balance FROM users WHERE user_id=?", (ref_id,))
                    invited, balance = await cur.fetchone()

                    cur = await db.execute("SELECT invited_today FROM invites WHERE user_id=?", (ref_id,))
                    today = (await cur.fetchone())[0]

                    await bot.send_message(
                        ref_id,
                        f"➕ +1 людина\n"
                        f"👥 Запрошено: {invited}\n"
                        f"📉 Залишилось: {max(0, DAILY_NORM - today)}\n"
                        f"💰 Баланс: {balance} грн"
                    )

    await message.answer("👋 Вітаю!", reply_markup=menu)


# 📊 Статистика
@dp.message(lambda msg: msg.text == "📊 Статистика")
async def stats(message: types.Message):
    user_id = message.from_user.id

    async with aiosqlite.connect(DB_NAME) as db:
        cur = await db.execute("SELECT invited, balance, day FROM users WHERE user_id=?", (user_id,))
        invited, balance, day = await cur.fetchone()

        cur = await db.execute("SELECT invited_today FROM invites WHERE user_id=?", (user_id,))
        today = (await cur.fetchone())[0]

    await message.answer(
        f"📊 Статистика:\n\n"
        f"👥 Запрошено: {invited}\n"
        f"📅 Робочих днів: {day}\n"
        f"📉 Залишилось: {max(0, DAILY_NORM - today)}\n"
        f"💰 Баланс: {balance} грн"
    )


# 💸 Вивід коштів
@dp.message(lambda msg: msg.text == "💸 Вивести кошти")
async def withdraw(message: types.Message):
    user_id = message.from_user.id

    async with aiosqlite.connect(DB_NAME) as db:
        cur = await db.execute("SELECT balance, day FROM users WHERE user_id=?", (user_id,))
        balance, day = await cur.fetchone()

    if balance >= MIN_BALANCE and day >= MIN_DAYS:
        await message.answer("✅ Заявка на вивід прийнята!")
    else:
        await message.answer(
            f"❌ Недостатньо умов:\n"
            f"Баланс: {balance}/{MIN_BALANCE}\n"
            f"Днів: {day}/{MIN_DAYS}"
        )


# 🔗 Реферальне посилання
@dp.message(lambda msg: msg.text == "🔗 Моє посилання")
async def link(message: types.Message):
    bot_info = await bot.get_me()
    await message.answer(f"https://t.me/{bot_info.username}?start={message.from_user.id}")


# 🔔 Нагадування
async def reminder():
    async with aiosqlite.connect(DB_NAME) as db:
        cur = await db.execute("SELECT user_id, invited_today FROM invites")
        users_list = await cur.fetchall()

        for user_id, today in users_list:
            if today < DAILY_NORM:
                try:
                    await bot.send_message(
                        user_id,
                        f"⚠️ Ти не виконав денну норму!\n"
                        f"Залишилось: {DAILY_NORM - today} людей"
                    )
                except:
                    pass


# 🌙 Скидання дня
async def reset_day():
    async with aiosqlite.connect(DB_NAME) as db:
        cur = await db.execute("SELECT user_id, invited_today FROM invites")
        users_list = await cur.fetchall()

        for user_id, today in users_list:
            if today >= DAILY_NORM:
                await db.execute("UPDATE users SET day = day + 1 WHERE user_id=?", (user_id,))
            await db.execute("UPDATE invites SET invited_today = 0 WHERE user_id=?", (user_id,))

        await db.commit()


# ⏰ Планувальник
scheduler = AsyncIOScheduler()
scheduler.add_job(reminder, "cron", hour=20)  # нагадування о 20:00
scheduler.add_job(reset_day, "cron", hour=0) # скидання о 00:00
scheduler.start()


# ▶️ Запуск бота
async def main():
    await init_db()
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
