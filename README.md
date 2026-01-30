# telegram-bot
import logging
from datetime import timedelta
from telegram import Update, ChatPermissions
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    ContextTypes,
)

TOKEN = "8567846209:AAEnyv7F_31q0mTDyii4i68QtXf1mmfz3fU"

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)

# ===== ХРАНИЛИЩА =====
nicknames = {}   # user_id: nickname
known_users = set()

# ===== ВСПОМОГАТЕЛЬНО =====
def is_admin(update: Update):
    member = update.effective_chat.get_member(update.effective_user.id)
    return member.status in ("administrator", "creator")

# ===== КОМАНДЫ =====
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    known_users.add(update.effective_user.id)
    await update.message.reply_text("✅ Админ-бот работает 24/7")

async def snick(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update):
        return await update.message.reply_text("❌ Только админ")

    if len(context.args) < 2 or not update.message.reply_to_message:
        return await update.message.reply_text("Используй: /snick <ник> (ответом на сообщение)")

    user = update.message.reply_to_message.from_user
    nick = context.args[0]
    nicknames[user.id] = nick
    await update.message.reply_text(f"✅ Ник {user.first_name} → {nick}")

async def rnick(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update):
        return await update.message.reply_text("❌ Только админ")

    if not update.message.reply_to_message:
        return await update.message.reply_text("Используй ответом на сообщение")

    user = update.message.reply_to_message.from_user
    nicknames.pop(user.id, None)
    await update.message.reply_text(f"🗑 Ник {user.first_name} удалён")

async def kick(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update):
        return await update.message.reply_text("❌ Только админ")

    if not update.message.reply_to_message:
        return await update.message.reply_text("Используй ответом на сообщение")

    user = update.message.reply_to_message.from_user
    await update.effective_chat.ban_member(user.id)
    await update.effective_chat.unban_member(user.id)
    await update.message.reply_text(f"👢 {user.first_name} кикнут")

async def mute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update):
        return await update.message.reply_text("❌ Только админ")

    if len(context.args) < 1 or not update.message.reply_to_message:
        return await update.message.reply_text("Используй: /mute <секунды> (ответом)")

    seconds = int(context.args[0])
    user = update.message.reply_to_message.from_user

    permissions = ChatPermissions(can_send_messages=False)
    until = timedelta(seconds=seconds)

    await update.effective_chat.restrict_member(
        user.id,
        permissions,
        until_date=update.message.date + until
    )

    await update.message.reply_text(f"🔇 {user.first_name} замучен на {seconds} сек.")

async def online(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = "🟢 Онлайн/известные пользователи:\n"
    for uid in known_users:
        text += f"- {nicknames.get(uid, uid)}\n"
    await update.message.reply_text(text)

async def all_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update):
        return await update.message.reply_text("❌ Только админ")

    if not context.args:
        return await update.message.reply_text("Используй: /all <сообщение>")

    msg = "📢 " + " ".join(context.args)
    for uid in known_users:
        try:
            await context.bot.send_message(uid, msg)
        except:
            pass

    await update.message.reply_text("✅ Рассылка отправлена")

# ===== ЗАПУСК =====
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("snick", snick))
app.add_handler(CommandHandler("rnick", rnick))
app.add_handler(CommandHandler("kick", kick))
app.add_handler(CommandHandler("mute", mute))
app.add_handler(CommandHandler("online", online))
app.add_handler(CommandHandler("all", all_cmd))

app.run_polling()
