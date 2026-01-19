import os, time, re, sqlite3, random, asyncio
from telegram import Update, ChatPermissions
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler,
    ContextTypes, filters
)
import openai

TOKEN = os.getenv("BOT_TOKEN")
OPENAI_KEY = os.getenv("OPENAI_KEY")
OWNER_ID = int(os.getenv("OWNER_ID") or 0)

openai.api_key = OPENAI_KEY

db = sqlite3.connect("bot.db", check_same_thread=False)
cur = db.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    warns INTEGER DEFAULT 0
)
""")
db.commit()

BAD_WORDS = ["spam", "scam", "fake"]
LINK_REGEX = r"(https?://|www\.)"
FLOOD_LIMIT = 5
FLOOD_TIME = 5
flood = {}
captcha_users = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Bot Online!")

async def welcome(update: Update, context: ContextTypes.DEFAULT_TYPE):
    for user in update.message.new_chat_members:
        a, b = random.randint(1,9), random.randint(1,9)
        captcha_users[user.id] = a + b
        await context.bot.restrict_chat_member(update.effective_chat.id, user.id, ChatPermissions(can_send_messages=False))
        await update.message.reply_text(f"CAPTCHA: {a} + {b} ? (60s)")
        await asyncio.sleep(60)
        if user.id in captcha_users:
            await context.bot.ban_chat_member(update.effective_chat.id, user.id)
            del captcha_users[user.id]

async def captcha_check(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    if uid in captcha_users and update.message.text.isdigit():
        if int(update.message.text) == captcha_users[uid]:
            del captcha_users[uid]
            await context.bot.restrict_chat_member(update.effective_chat.id, uid, ChatPermissions(can_send_messages=True))
            await update.message.reply_text("Verified!")

async def anti_spam(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.lower()
    if re.search(LINK_REGEX, text):
        await update.message.delete()
        return
    if any(word in text for word in BAD_WORDS):
        await update.message.delete()

async def anti_flood(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    now = time.time()
    flood.setdefault(uid, []).append(now)
    flood[uid] = [t for t in flood[uid] if now - t < FLOOD_TIME]
    if len(flood[uid]) > FLOOD_LIMIT:
        await context.bot.restrict_chat_member(update.effective_chat.id, uid, ChatPermissions(can_send_messages=False))

async def ai_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_text = update.message.text
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role":"system","content":"You are a helpful Telegram bot assistant."},
                {"role":"user","content":user_text}
            ]
        )
        reply = response["choices"][0]["message"]["content"]
        await update.message.reply_text(reply)
    except:
        await update.message.reply_text("AI Error.")

def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, captcha_check))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, anti_flood))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, anti_spam))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, ai_reply))
    app.run_polling()

if __name__ == "__main__":
    main()
