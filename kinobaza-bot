import asyncio
import logging
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart, Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup, KeyboardButton
import aiosqlite

# --- SOZLAMALAR ---
BOT_TOKEN = "8634039492:AAF_RmClS3qUxkX1QtuS1ABcvbhGwUyFfEE"  
ADMIN_ID = 1316308230          
KANAL_ID = -1002271887321  # Kanal ID'si (masalan: -100123456789)
KANAL_LINK = "https://t.me/your_channel_username"  # Majburiy obuna kanali havolasi
CLICK_PAYMENT_URL = "https://click.uz/" # Click to'lov havolasi

logging.basicConfig(level=logging.INFO)

class MovieState(StatesGroup):
    waiting_for_file = State()
    waiting_for_name = State()
    waiting_for_code = State()

class ReqState(StatesGroup):
    waiting_for_ad = State()

class SupportState(StatesGroup):
    waiting_for_message = State()

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
DB_NAME = "kinobot_fast.db"

# --- BAZANI REJALASHTIRISH ---
async def init_db():
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY, 
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS movies (
                code TEXT PRIMARY KEY, 
                name TEXT, 
                file_id TEXT, 
                file_type TEXT
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS requests (
                id INTEGER PRIMARY KEY AUTOINCREMENT, 
                user_id INTEGER, 
                text TEXT
            )
        """)
        await db.commit()

# --- TUGMALAR ---
def get_channel_keyboard():
    buttons = [
        [InlineKeyboardButton(text="📢 Telegram kanalga obuna bo'lish", url=KANAL_LINK)],
        [InlineKeyboardButton(text="✅ Obunani tekshirish", callback_data="check_subs")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def get_main_keyboard():
    buttons = [
        [KeyboardButton(text="💳 To'lov qilish"), KeyboardButton(text="✍️ Adminga murojaat")]
    ]
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True)

def get_admin_keyboard():
    buttons = [
        [KeyboardButton(text="📊 Statistika"), KeyboardButton(text="📈 Statistika 2")],
        [KeyboardButton(text="🎬 Kino qo'shish"), KeyboardButton(text="📢 Reklama yuborish")],
        [KeyboardButton(text="📝 So'ralgan kinolar"), KeyboardButton(text="🔙 Asosiy menyu")]
    ]
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True)

# --- MAJBURITY OBUNANI TEKSHIRISH ---
async def check_subscription(user_id: int) -> bool:
    if user_id == ADMIN_ID:
        return True
    try:
        member = await bot.get_chat_member(chat_id=KANAL_ID, user_id=user_id)
        if member.status in ["member", "administrator", "creator"]:
            return True
        return False
    except Exception as e:
        logging.error(f"Tekshirishda xatolik: {e}")
        return False

# --- START VA ASOSIY BUYRUKLAR ---
@dp.message(CommandStart())
async def start_cmd(message: types.Message):
    user_id = message.from_user.id
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        await db.commit()
    
    if await check_subscription(user_id):
        await message.answer(
            "Xush kelibsiz! Kino kodini kiriting yoki quyidagi tugmalardan foydalaning. 🎬", 
            reply_markup=get_main_keyboard()
        )
    else:
        await message.answer(
            "⚠️ Botdan foydalanish uchun rasmiy kanalimizga a'zo bo'ling va **'Obunani tekshirish'** tugmasini bosing:",
            reply_markup=get_channel_keyboard()
        )

@dp.callback_query(F.data == "check_subs")
async def check_callback(callback: types.CallbackQuery):
    user_id = callback.from_user.id
    if await check_subscription(user_id):
        try:
            await callback.message.delete()
        except Exception:
            pass
        await callback.message.answer(
            "✅ Obuna tasdiqlandi! Kino kodini yuborishingiz mumkin. 🎬", 
            reply_markup=get_main_keyboard()
        )
    else:
        await callback.answer("❌ Siz hali kanalga obuna bo'lmadingiz! A'zo bo'lib qayta urining.", show_alert=True)

# --- CLICK TO'LOV ---
@dp.message(F.text == "💳 To'lov qilish")
async def payment_cmd(message: types.Message):
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="💳 Click orqali tezkor to'lov", url=CLICK_PAYMENT_URL)],
            [InlineKeyboardButton(text="✅ To'lov qildim (Adminga yuborish)", callback_data="paid_confirm")]
        ]
    )
    await message.answer(
        "🔹 **Click orqali to'lov qilish:**\n\n"
        "1. Pastdagi **'Click orqali tezkor to'lov'** tugmasini bosing.\n"
        "2. To'lovni amalga oshirgach **'To'lov qildim'** tugmasini bosing.", 
        reply_markup=keyboard
    )

@dp.callback_query(F.data == "paid_confirm")
async def paid_confirm_callback(callback: types.CallbackQuery):
    await callback.answer("Xabaringiz adminga yuborildi! Chekni adminga jo'natishingiz mumkin.", show_alert=True)
    await bot.send_message(
        ADMIN_ID, 
        f"💳 **Yangi to'lov bildirishnomasi!**\nFoydalanuvchi: @{callback.from_user.username or 'Mavjud emas'} (ID: {callback.from_user.id})"
    )

# --- ADMINGA MUROJAAT ---
@dp.message(F.text == "✍️ Adminga murojaat")
async def support_cmd(message: types.Message, state: FSMContext):
    await message.answer("Adminga yo'llamoqchi bo'lgan xabaringizni yuboring:")
    await state.set_state(SupportState.waiting_for_message)

@dp.message(SupportState.waiting_for_message)
async def process_support_message(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    username = f"@{message.from_user.username}" if message.from_user.username else f"ID: {user_id}"
    
    forward_text = f"📩 **Yangi murojaat!**\nFoydalanuvchi: {username} ({user_id})\n\nXabar:\n{message.text}"
    try:
        await bot.send_message(ADMIN_ID, forward_text)
        await message.answer("✅ Xabaringiz adminga yetkazildi!", reply_markup=get_main_keyboard())
    except Exception:
        await message.answer("❌ Xatolik yuz berdi, keyinroq qayta urinib ko'ring.", reply_markup=get_main_keyboard())
    await state.clear()

# --- ADMIN PANEL BUYRUKLARI ---
@dp.message(Command("admin"))
async def admin_panel(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        await message.answer("🛠 Admin panelga xush kelibsiz!", reply_markup=get_admin_keyboard())

@dp.message(F.text == "🔙 Asosiy menyu")
async def back_to_main(message: types.Message):
    await message.answer("Asosiy menyuga qaytdingiz.", reply_markup=get_main_keyboard())

# 1. STATISTIKA (Jami foydalanuvchilar va kinolar)
@dp.message(F.text == "📊 Statistika")
async def show_stats(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        async with aiosqlite.connect(DB_NAME) as db:
            async with db.execute("SELECT COUNT(*) FROM users") as cursor:
                u_count = (await cursor.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM movies") as cursor:
                m_count = (await cursor.fetchone())[0]
        await message.answer(f"📊 **Umumiy statistika:**\n\n👥 Jami obunachilar (Start bosganlar): **{u_count}** ta\n🎬 Jami kinolar: **{m_count}** ta")

# 2. STATISTIKA 2 (Vaqtlar bo'yicha)
@dp.message(F.text == "📈 Statistika 2")
async def show_advanced_stats(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        now = datetime.now()
        today_start = now.replace(hour=0, minute=0, second=0, microsecond=0).strftime('%Y-%m-%d %H:%M:%S')
        week_start = (now - timedelta(days=7)).strftime('%Y-%m-%d %H:%M:%S')
        month_start = (now - timedelta(days=30)).strftime('%Y-%m-%d %H:%M:%S')

        async with aiosqlite.connect(DB_NAME) as db:
            async with db.execute("SELECT COUNT(*) FROM users WHERE created_at >= ?", (today_start,)) as c:
                today_count = (await c.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM users WHERE created_at >= ?", (week_start,)) as c:
                week_count = (await c.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM users WHERE created_at >= ?", (month_start,)) as c:
                month_count = (await c.fetchone())[0]

        msg = (
            f"📈 **Vaqt bo'yicha statistika:**\n\n"
            f"☀️ Bugun start bosganlar: **{today_count}** ta\n"
            f"📅 Oxirgi 1 haftada: **{week_count}** ta\n"
            f"📆 Oxirgi 1 oyda: **{month_count}** ta"
        )
        await message.answer(msg)

# 3. KINO QO'SHISH
@dp.message(F.text == "🎬 Kino qo'shish")
async def add_movie_start(message: types.Message, state: FSMContext):
    if message.from_user.id == ADMIN_ID:
        await message.answer("1️⃣ Kinoni video yoki hujjat (file) ko'rinishida yuboring:")
        await state.set_state(MovieState.waiting_for_file)

@dp.message(MovieState.waiting_for_file, F.video | F.document)
async def process_movie_file(message: types.Message, state: FSMContext):
    file_id = message.video.file_id if message.video else message.document.file_id
    file_type = "video" if message.video else "document"
    await state.update_data(file_id=file_id, file_type=file_type)
    await message.answer("2️⃣ Endi kino nomini kiriting:")
    await state.set_state(MovieState.waiting_for_name)

@dp.message(MovieState.waiting_for_name)
async def process_movie_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("3️⃣ Endi kino uchun kod kiriting (masalan: 102):")
    await state.set_state(MovieState.waiting_for_code)

@dp.message(MovieState.waiting_for_code)
async def process_movie_code(message: types.Message, state: FSMContext):
    code = message.text.strip()
    data = await state.get_data()
    async with aiosqlite.connect(DB_NAME) as db:
        try:
            await db.execute("INSERT INTO movies (code, name, file_id, file_type) VALUES (?, ?, ?, ?)", 
                           (code, data['name'], data['file_id'], data['file_type']))
            await db.commit()
            await message.answer(f"✅ Kino muvaffaqiyatli saqlandi!\n\n🎬 Nomi: {data['name']}\n🔑 Kodi: {code}")
        except aiosqlite.IntegrityError:
            await message.answer("❌ Bu kod bazada mavjud! Boshqa kod kiriting.")
    await state.clear()

# 4. TEZKOR REKLAMA TARQATISH
@dp.message(F.text == "📢 Reklama yuborish")
async def start_ad(message: types.Message, state: FSMContext):
    if message.from_user.id == ADMIN_ID:
        await message.answer("📢 Reklama postini yuboring (Rasm, matn, video yoki havola shaklida):")
        await state.set_state(ReqState.waiting_for_ad)

@dp.message(ReqState.waiting_for_ad)
async def send_ad_to_all(message: types.Message, state: FSMContext):
    async with aiosqlite.connect(DB_NAME) as db:
        async with db.execute("SELECT user_id FROM users") as cursor:
            users = await cursor.fetchall()
    
    await message.answer("⚡️ Reklama tarqatilishi boshlandi...")
    count = 0
    for user in users:
        try:
            await message.copy_to(chat_id=user[0])
            count += 1
            await asyncio.sleep(0.03)  # Maksimal tezkor va xavfsiz yuborish
        except Exception:
            pass
    await message.answer(f"✅ Reklama **{count}** ta foydalanuvchiga muvaffaqiyatli yetkazildi!")
    await state.clear()

@dp.message(F.text == "📝 So'ralgan kinolar")
async def show_requests(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        async with aiosqlite.connect(DB_NAME) as db:
            async with db.execute("SELECT text, COUNT(text) FROM requests GROUP BY text ORDER BY COUNT(text) DESC LIMIT 20") as cursor:
                reqs = await cursor.fetchall()
        if not reqs:
            await message.answer("Hozircha hech qanday kino so'ralmagan.")
            return
        text = "📝 **Eng ko'p so'ralgan kodlar:**\n\n"
        for r in reqs:
            text += f"🔹 Kod: {r[0]} — {r[1]} marta\n"
        await message.answer(text)

# --- KINO QIDIRISH ---
@dp.message(F.text)
async def search_movie(message: types.Message):
    user_id = message.from_user.id
    if not await check_subscription(user_id):
        await message.answer(
            "⚠️ Botdan foydalanish uchun rasmiy kanalimizga a'zo bo'ling va **'Obunani tekshirish'** tugmasini bosing:",
            reply_markup=get_channel_keyboard()
        )
        return
    
    code = message.text.strip()
    async with aiosqlite.connect(DB_NAME) as db:
        async with db.execute("SELECT name, file_id, file_type FROM movies WHERE code = ?", (code,)) as cursor:
            movie = await cursor.fetchone()
        
        if movie:
            m_name, m_file_id, m_file_type = movie
            caption_text = f"🎬 **Kino nomi:** {m_name}\n🔑 **Kodi:** {code}"
            if m_file_type == "video":
                await message.answer_video(video=m_file_id, caption=caption_text)
            else:
                await message.answer_document(document=m_file_id, caption=caption_text)
        else:
            try:
                await db.execute("INSERT INTO requests (user_id, text) VALUES (?, ?)", (user_id, code))
                await db.commit()
            except Exception:
                pass
            await message.answer("❌ Afsuski, bu kod bilan kino topilmadi. So'rovingiz adminga yetkazildi!")

async def main():
    await init_db()
    print("Bot muvaffaqiyatli ishga tushdi!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
