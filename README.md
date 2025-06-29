# bot.py  —  بوت الهوية الدائمة
# ---------------------------------
# متطلبات: pip install aiogram==2.25.1
# شغِّل:   python bot.py

import json, os, random, string, time, asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.utils import executor

# ----- إعدادات ثابتة -----
BOT_TOKEN = "7759174727:AAEXryXRMxEatIGOch_gglkLWcJJXajG3lI"   # توكن البوت
OWNER_ID  = 5447365946                                          # آيدي المالك
DB_FILE   = "users.json"                                        # قاعدة البيانات

# ----- تهيئة البوت -----
bot = Bot(BOT_TOKEN, parse_mode="HTML")
dp  = Dispatcher(bot)

# ----- دوال مساعدة -----
def load_db():
    if not os.path.exists(DB_FILE):
        return {}
    with open(DB_FILE, "r", encoding="utf8") as f:
        return json.load(f)

def save_db(db):
    with open(DB_FILE, "w", encoding="utf8") as f:
        json.dump(db, f, ensure_ascii=False, indent=2)

def gen_ident():
    return "ZN-" + "".join(random.choices(string.digits, k=6))

# ----- /start -----
@dp.message_handler(commands=["start"])
async def cmd_start(msg: types.Message):
    db  = load_db()
    uid = str(msg.from_user.id)

    # إذا المستخدم موجود من قبل
    if uid in db:
        ident = db[uid]["ident"]
        await msg.answer(f"⚠️ هويتك الدائمة:\n<code>{ident}</code>")
        return

    # مستخدم جديد
    ident = gen_ident()
    db[uid] = {"ident": ident, "ts": time.time()}
    save_db(db)

    await msg.answer(
        f"👋 هلا بيك <b>{msg.from_user.first_name}</b>\n"
        f"هويتك داخل تطبيقاتنا هي:\n<code>{ident}</code>\n"
        "احفظها زين، ما راح تتغيّر أبدًا."
    )
    # إشعار للمالك
    await bot.send_message(
        OWNER_ID,
        f"🆕 مستخدم جديد:\nID: <code>{uid}</code>\n"
        f"Tag: @{msg.from_user.username}\n"
        f"Ident: <code>{ident}</code>"
    )

# ----- /myid  (يعرض الهوية للمستخدم) -----
@dp.message_handler(commands=["myid"])
async def cmd_myid(msg: types.Message):
    db = load_db()
    uid = str(msg.from_user.id)
    if uid in db:
        await msg.answer(f"🔖 هويتك:\n<code>{db[uid]['ident']}</code>")
    else:
        await msg.answer("❌ ما مسجل بعد – أكتب /start أولاً")

# ----- /stats  (إحصائيات) -----
@dp.message_handler(commands=["stats"])
async def cmd_stats(msg: types.Message):
    if msg.from_user.id != OWNER_ID:
        return
    db = load_db()
    await msg.reply(f"📊 عدد المستخدمين الكلي: <b>{len(db)}</b>")

# ----- /broadcast نص الرسالة  (إرسال جماعي) -----
@dp.message_handler(commands=["broadcast"])
async def cmd_broadcast(msg: types.Message):
    if msg.from_user.id != OWNER_ID:
        return
    text = msg.get_args()
    if not text:
        await msg.reply("استعمال صحيح:\n/broadcast نص_الرسالة")
        return

    db = load_db()
    sent, fail = 0, 0
    for uid in db.keys():
        try:
            await bot.send_message(uid, text)
            sent += 1
            await asyncio.sleep(0.05)    # تخفيف الحمل
        except:
            fail += 1
    await msg.reply(f"تم الإرسال ✅{sent} | فشل ❌{fail}")

# ----- تشغيل البوت -----
if __name__ == "__main__":
    print("Bot is running…")
    executor.start_polling(dp, skip_updates=True)
