# dep
Официальный Telegram-бот для акции копчёного курта от «Гормолзавод Кокшетау». Бот позволяет регистрировать пользователей, принимать уникальные коды с упаковок, начислять баллы и билеты для участия в еженедельных и финальных розыгрышах, а также обменивать баллы на пополнение мобильного баланса.

## Стек технологий
* **Язык:** Python 3.9+
* **Библиотека Telegram:** aiogram 3.x
* **База данных:** Supabase (PostgreSQL)

## Структура проекта
* `bot.py` — основной файл с логикой бота.
* `schema.sql` — SQL-скрипт для создания необходимых таблиц и функций в Supabase.
* `requirements.txt` — зависимости проекта.
* `.env.example` — шаблон для переменных окружения.

## Инструкция по установке

### 1. Настройка базы данных (Supabase)
1. Зарегистрируйтесь на [Supabase](https://supabase.com) и создайте новый проект.
2. Перейдите в раздел **SQL Editor**.
3. Скопируйте содержимое файла `schema.sql` и выполните его. Это создаст таблицы `users`, `codes`, `reward_claims` и функцию `get_total_claims()`.
4. В настройках проекта (Project Settings -> API) скопируйте `Project URL` и `service_role secret`.

### 2. Настройка бота
1. Клонируйте репозиторий:
   `git clone https://github.com/ВАШ_АККАУНТ/ВАШ_РЕПОЗИТОРИЙ.git`
2. Перейдите в папку проекта:
   `cd ВАШ_РЕПОЗИТОРИЙ`
3. Создайте виртуальное окружение и активируйте его:
   `python3 -m venv venv`
   `source venv/bin/activate` (для Mac/Linux) или `venv\Scripts\activate` (для Windows).
4. Установите зависимости:
   `pip install -r requirements.txt`
5. Создайте файл `.env` на основе шаблона и заполните его вашими ключами:
   ```env
   BOT_TOKEN=ваш_токен_от_botfather
   SUPABASE_URL=ваш_project_url_из_supabase
   SUPABASE_SERVICE_KEY=ваш_service_role_key_из_supabase
3. Запуск бота
python3 bot.py

---

### Файл `bot.py`
Скопируйте этот код и сохраните его в файл `bot.py`:

```python
import asyncio
import os
import re

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.types import Message, ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove, InlineKeyboardMarkup, InlineKeyboardButton, CallbackQuery
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from supabase import create_client

bot = Bot(
    token=os.environ["BOT_TOKEN"],
    default=DefaultBotProperties(parse_mode=ParseMode.HTML)
)
dp = Dispatcher()
db = create_client(os.environ["SUPABASE_URL"], os.environ["SUPABASE_SERVICE_KEY"])

POINTS_PER_CODE = 10
REWARD_COST = 20
MAX_REWARDS = 500

# Главное меню
main_kb = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="🎁 Ввести код"), KeyboardButton(text="👤 Мой баланс")],
        [KeyboardButton(text="🏆 Призы и победители"), KeyboardButton(text="ℹ️ Правила акции")]
    ],
    resize_keyboard=True
)

# Состояния регистрации
class RegState(StatesGroup):
    rules_agree = State()
    name = State()
    phone = State()
    city = State()
    age = State()
    promo = State()
    
# Состояние ввода кода
class CodeState(StatesGroup):
    waiting_for_code = State()

def get_user(telegram_id: int):
    res = db.table("users").select("*").eq("telegram_id", telegram_id).execute()
    return res.data[0] if res.data else None

@dp.message(Command("start"))
async def start_cmd(m: Message, state: FSMContext):
    user = get_user(m.from_user.id)
    if user:
        await m.answer("👋 С возвращением! Выберите действие в меню ниже.", reply_markup=main_kb)
        return

    text = (
        "👋 Добро пожаловать в официальный бот акции копчёного курта от «Гормолзавод Кокшетау»!\n\n"
        "Покупайте акционный курт, находите уникальные коды внутри упаковки и участвуйте в розыгрышах призов.\n\n"
        "📌 <b>За каждый код вы получаете:</b>\n"
        "• 10 баллов\n"
        "• 1 билет на ближайший еженедельный розыгрыш\n\n"
        "📌 <b>Дополнительно:</b>\n"
        "• Собрав 20 баллов, обменяйте их на 500 ₸ на баланс (всего 500 призов на акцию!).\n"
        "• Каждые 5 кодов = 1 билет на финальный розыгрыш суперприза.\n\n"
        "Чтобы начать, ознакомьтесь с Правилами акции и Политикой обработки данных."
    )
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Согласен с правилами", callback_data="agree_rules")]
    ])
    await m.answer(text, reply_markup=kb)

@dp.callback_query(F.data == "agree_rules")
async def agree_rules_cb(c: CallbackQuery, state: FSMContext):
    await c.message.edit_reply_markup(reply_markup=None)
    await state.set_state(RegState.name)
    await c.message.answer("Отлично! Для участия нам нужно познакомиться. Как к вам обращаться?", reply_markup=ReplyKeyboardRemove())

@dp.message(RegState.name)
async def process_name(m: Message, state: FSMContext):
    await state.update_data(name=m.text)
    await state.set_state(RegState.phone)
    kb = ReplyKeyboardMarkup(keyboard=[[KeyboardButton(text="📱 Отправить номер", request_contact=True)]], resize_keyboard=True)
    await m.answer("Пожалуйста, поделитесь вашим номером телефона. Он понадобится для пополнения баланса и связи в случае выигрыша.", reply_markup=kb)

@dp.message(RegState.phone, F.contact)
async def process_phone_contact(m: Message, state: FSMContext):
    await state.update_data(phone=m.contact.phone_number)
    await state.set_state(RegState.city)
    await m.answer("Укажите ваш город:", reply_markup=ReplyKeyboardRemove())
    
@dp.message(RegState.phone)
async def process_phone_text(m: Message, state: FSMContext):
    await state.update_data(phone=m.text)
    await state.set_state(RegState.city)
    await m.answer("Укажите ваш город:", reply_markup=ReplyKeyboardRemove())

@dp.message(RegState.city)
async def process_city(m: Message, state: FSMContext):
    await state.update_data(city=m.text)
    await state.set_state(RegState.age)
    kb = ReplyKeyboardMarkup(keyboard=[
        [KeyboardButton(text="До 18"), KeyboardButton(text="18-25")],
        [KeyboardButton(text="26-35"), KeyboardButton(text="36-45")],
        [KeyboardButton(text="Старше 45")]
    ], resize_keyboard=True)
    await m.answer("Укажите вашу возрастную группу:", reply_markup=kb)

@dp.message(RegState.age)
async def process_age(m: Message, state: FSMContext):
    await state.update_data(age=m.text)
    await state.set_state(RegState.promo)
    kb = ReplyKeyboardMarkup(keyboard=[
        [KeyboardButton(text="Да, хочу"), KeyboardButton(text="Нет, спасибо")]
    ], resize_keyboard=True)
    await m.answer("Хотите ли вы получать уведомления о новых продуктах и акциях в будущем?", reply_markup=kb)

@dp.message(RegState.promo)
async def process_promo(m: Message, state: FSMContext):
    data = await state.get_data()
    agreed = True if m.text == "Да, хочу" else False
    
    # Сохраняем в БД
    db.table("users").insert({
        "telegram_id": m.from_user.id,
        "name": data["name"],
        "phone": data["phone"],
        "city": data["city"],
        "age_group": data["age"],
        "agreed_to_promo": agreed
    }).execute()
    
    await state.clear()
    await m.answer("🎉 Регистрация завершена! Теперь вы можете вводить коды с упаковок курта.", reply_markup=main_kb)

# ---- Ввод кода ----
@dp.message(F.text == "🎁 Ввести код")
async def enter_code_btn(m: Message, state: FSMContext):
    if not get_user(m.from_user.id):
        return await m.answer("Сначала нажмите /start для регистрации.")
    await state.set_state(CodeState.waiting_for_code)
    await m.answer("Пожалуйста, отправьте уникальный код с упаковки (например, ABC12345):", reply_markup=ReplyKeyboardRemove())

@dp.message(CodeState.waiting_for_code)
async def process_code(m: Message, state: FSMContext):
    code_text = m.text.strip().upper()
    await state.clear()
    
    user = get_user(m.from_user.id)
    if not user:
        return await m.answer("Ошибка: пользователь не найден. Нажмите /start", reply_markup=main_kb)
    
    # Проверяем код в БД
    code_res = db.table("codes").select("*").eq("code", code_text).execute()
    if not code_res.data:
        return await m.answer("❌ Код не найден. Пожалуйста, проверьте, правильно ли вы его ввели, и попробуйте еще раз.", reply_markup=main_kb)
    
    code_obj = code_res.data[0]
    if code_obj.get("used"):
        return await m.answer("⚠️ Этот код уже был зарегистрирован ранее. Каждый уникальный код можно использовать только один раз.", reply_markup=main_kb)
        
    # Отмечаем код как использованный
    db.table("codes").update({"used": True, "used_by": m.from_user.id, "used_at": "now()"}).eq("code", code_text).execute()
    
    # Начисляем пользователю
    new_points = user["points"] + POINTS_PER_CODE
    new_weekly = user["weekly_tickets"] + 1
    new_total = user["total_codes_entered"] + 1
    
    db.table("users").update({
        "points": new_points,
        "weekly_tickets": new_weekly,
        "total_codes_entered": new_total
    }).eq("telegram_id", m.from_user.id).execute()
    
    msg = f"✅ Код успешно принят!\nВам начислено:\n• {POINTS_PER_CODE} баллов\n• 1 билет в еженедельный розыгрыш\n"
    if new_total % 5 == 0:
        msg += "\n🎉 Поздравляем! Вы получили 1 билет на финальный розыгрыш!\n"
    msg += "\nПроверить свои шансы можно в разделе «👤 Мой баланс»."
    
    await m.answer(msg, reply_markup=main_kb)


# ---- Баланс и Обмен ----
@dp.message(F.text == "👤 Мой баланс")
async def show_balance(m: Message):
    user = get_user(m.from_user.id)
    if not user:
        return await m.answer("Сначала нажмите /start")
        
    final_tickets = user["total_codes_entered"] // 5
    rem = 5 - (user["total_codes_entered"] % 5)
    
    text = (
        f"👤 <b>Ваш профиль:</b> {user['name']} ({user['city']})\n\n"
        f"💰 <b>Ваши баллы:</b> {user['points']}\n"
        f"🎟 <b>Билетов на еженедельный розыгрыш:</b> {user['weekly_tickets']}\n"
        f"🎫 <b>Билетов на финальный розыгрыш:</b> {final_tickets} <i>(осталось ввести {rem} кода до следующего билета)</i>\n\n"
        "<i>Накопленные баллы можно обменять на пополнение баланса, при этом ваши билеты для розыгрышей сохраняются!</i>"
    )
    
    kb = None
    if user['points'] >= REWARD_COST:
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text=f"🔄 Обменять {REWARD_COST} баллов на 500 ₸", callback_data="claim_reward")]
        ])
        
    await m.answer(text, reply_markup=kb or main_kb)

@dp.callback_query(F.data == "claim_reward")
async def claim_reward_cb(c: CallbackQuery):
    user = get_user(c.from_user.id)
    if user['points'] < REWARD_COST:
        return await c.answer("Недостаточно баллов!", show_alert=True)
        
    # Проверяем лимит
    claims_res = db.rpc("get_total_claims", {}).execute()
    total_claims = claims_res.data if claims_res.data else 0
    
    if total_claims >= MAX_REWARDS:
        await c.message.edit_text("😔 К сожалению, фонд гарантированных наград (500 пополнений) уже исчерпан. Но ваши билеты продолжают участвовать в еженедельных и финальном розыгрышах! Желаем удачи!")
        return
        
    remains = MAX_REWARDS - total_claims
    text = (
        f"Внимание! Количество гарантированных пополнений ограничено (осталось: {remains} из {MAX_REWARDS}).\n"
        f"С вашего баланса будет списано {REWARD_COST} баллов. Ваши билеты для участия в розыгрышах <b>не сгорают</b>.\n\n"
        f"Перевести 500 ₸ на ваш номер {user['phone']}?"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Подтвердить", callback_data="confirm_claim")],
        [InlineKeyboardButton(text="❌ Отмена", callback_data="cancel_claim")]
    ])
    await c.message.edit_text(text, reply_markup=kb)

@dp.callback_query(F.data == "confirm_claim")
async def confirm_claim_cb(c: CallbackQuery):
    user = get_user(c.from_user.id)
    if user['points'] < REWARD_COST:
        return await c.answer("Недостаточно баллов!", show_alert=True)
        
    claims_res = db.rpc("get_total_claims", {}).execute()
    total_claims = claims_res.data if claims_res.data else 0
    if total_claims >= MAX_REWARDS:
        return await c.message.edit_text("😔 К сожалению, фонд исчерпан, пока вы принимали решение.")
        
    # Списываем баллы и создаем заявку
    db.table("users").update({"points": user['points'] - REWARD_COST}).eq("telegram_id", c.from_user.id).execute()
    db.table("reward_claims").insert({"telegram_id": c.from_user.id}).execute()
    
    await c.message.edit_text("✅ Заявка принята! 500 ₸ поступят на ваш баланс в ближайшее время. С вашего счёта списано 20 баллов.")

@dp.callback_query(F.data == "cancel_claim")
async def cancel_claim_cb(c: CallbackQuery):
    await c.message.delete()
    await c.answer("Обмен отменен")

# ---- Статика ----
@dp.message(F.text == "🏆 Призы и победители")
async def show_prizes(m: Message):
    text = (
        "🏆 <b>Призы и победители</b>\n\n"
        "Каждую неделю мы разыгрываем призы стоимостью до 25 000 ₸! В конце акции состоится розыгрыш суперприза до 150 000 ₸.\n\n"
        "Списки победителей будут публиковаться здесь после каждого розыгрыша."
    )
    await m.answer(text, reply_markup=main_kb)

@dp.message(F.text == "ℹ️ Правила акции")
async def show_rules(m: Message):
    text = "ℹ️ Подробные правила акции доступны по ссылке: [Ссылка на правила будет добавлена организатором]"
    await m.answer(text, reply_markup=main_kb)

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
Файл schema.sql (Для Supabase и GitHub)
Этот файл тоже стоит добавить в GitHub, чтобы история изменений базы данных сохранялась:

-- Пользователи и их анкетные данные
create table if not exists users (
    telegram_id bigint primary key,
    name text not null,
    phone text not null,
    city text not null,
    age_group text not null,
    agreed_to_promo boolean default false,
    
    -- Балансы
    points int default 0,
    weekly_tickets int default 0,
    total_codes_entered int default 0, -- для подсчета финальных билетов (каждые 5 = 1 билет)
    
    created_at timestamptz default now()
);

-- База уникальных кодов с упаковки
create table if not exists codes (
    code text primary key,
    used boolean default false,
    used_by bigint references users(telegram_id),
    used_at timestamptz
);

-- Таблица для учета заявок на выплату 500 тенге
create table if not exists reward_claims (
    id serial primary key,
    telegram_id bigint references users(telegram_id) not null,
    amount int default 500,
    status text default 'pending', -- pending, processed, failed
    claimed_at timestamptz default now()
);

alter table users enable row level security;
alter table codes enable row level security;
alter table reward_claims enable row level security;

-- Функция для проверки общего лимита наград (не больше 500 штук)
create or replace function get_total_claims()
returns bigint
language sql
security definer
as $$
  select count(*) from reward_claims;
$$;
Файл requirements.txt
aiogram==3.15.0
supabase==2.9.0
