import os
import sqlite3
from datetime import datetime, timedelta
from PIL import Image, ImageDraw, ImageFont
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes

TOKEN = os.getenv("TELEGRAM_TOKEN")

def init_db():
    conn = sqlite3.connect("tracker.db")
    c = conn.cursor()
    c.execute("CREATE TABLE IF NOT EXISTS habits (date TEXT PRIMARY KEY, status TEXT)")
    conn.commit()
    conn.close()

def set_status(date, status):
    conn = sqlite3.connect("tracker.db")
    c = conn.cursor()
    c.execute("REPLACE INTO habits (date, status) VALUES (?,?)", (date, status))
    conn.commit()
    conn.close()

def get_last_days(days=7):
    conn = sqlite3.connect("tracker.db")
    c = conn.cursor()
    end = datetime.now()
    start = end - timedelta(days=days-1)
    c.execute("SELECT date, status FROM habits WHERE date BETWEEN? AND?",
              (start.strftime("%Y-%m-%d"), end.strftime("%Y-%m-%d")))
    data = dict(c.fetchall())
    conn.close()

    result = []
    for i in range(days):
        d = (start + timedelta(days=i)).strftime("%Y-%m-%d")
        result.append((d, data.get(d, "✗")))
    return result

def make_table_image(data):
    width, height = 800, 100 + len(data)*60
    img = Image.new("RGB", (width, height), "white")
    draw = ImageDraw.Draw(img)

    try:
        font = ImageFont.truetype("DejaVuSans.ttf", 32)
    except:
        font = ImageFont.load_default()

    draw.text((20, 20), "Күнделікті Трекер", fill="black", font=font)
    y = 80
    for date, status in data:
        day = datetime.strptime(date, "%Y-%m-%d").strftime("%d.%m %A")
        color = "green" if status == "✓" else "red"
        draw.text((50, y), day, fill="black", font=font)
        draw.text((600, y), status, fill=color, font=font)
        y += 60

    path = "/tmp/table.png"
    img.save(path)
    return path

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    today = datetime.now().strftime("%Y-%m-%d")
    keyboard = [
        [InlineKeyboardButton("✓ Орындадым", callback_data=f"yes_{today}")],
        [InlineKeyboardButton("✗ Орындамадым", callback_data=f"no_{today}")]
    ]
    await update.message.reply_text(
        f"Бүгін {datetime.now().strftime('%d.%m.%Y')} үшін белгіле:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data.split("_")
    status = "✓" if data[0] == "yes" else "✗"
    date = data[1]
    set_status(date, status)
    await query.edit_message_text(f"{date} үшін белгіленді: {status}")

async def send_report(context: ContextTypes.DEFAULT_TYPE):
    chat_id = context.job.chat_id
    data = get_last_days(7)
    img_path = make_table_image(data)
    await context.bot.send_photo(chat_id, photo=open(img_path, "rb"),
                                 caption="Соңғы 7 күннің есебі")

async def history(update: Update, context: ContextTypes.DEFAULT_TYPE):
    data = get_last_days(30)
    img_path = make_table_image(data)
    await update.message.reply_photo(photo=open(img_path, "rb"),
                                     caption="Соңғы 30 күн")

if _name_ == "_main_":
    init_db()
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("history", history))
    app.add_handler(CallbackQueryHandler(button))
    app.job_queue.run_daily(send_report, time=datetime.strptime("00:00", "%H:%M").time())
    app.run_polling()
