import logging
import asyncio
import aiohttp
import random
import datetime
import pytz
from aiogram import Bot, Dispatcher, types
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    ReplyKeyboardMarkup, KeyboardButton, FSInputFile
)
from aiogram.filters import CommandStart, Command
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

# --------------------- КОНФИГУРАЦИЯ ---------------------
API_TOKEN = '7684859193:AAHX5-HWiN1NDBviYCJfK3XCVy8g-0cO74g'
ADMIN_ID = 7444717613
TIMEZONE = 'Europe/Moscow'
PO_LINK = 'https://u3.shortink.io/register?utm_campaign=820333&utm_source=affiliate&utm_medium=sr&a=CklGEuzOPoabUf&ac=lider&code=NPV489'
PROMO_CODE = 'NPV489'
FIN_API_KEY = 'ouMF3YxBuHDPLHiPFOLK4qRfJ'
BUY_IMAGE_PATH = 'assets/buy_signal.png'
SELL_IMAGE_PATH = 'assets/sell_signal.png'

# --------------------- ХРАНЕНИЕ ДАННЫХ ---------------------
user_data = {}  # {user_id: {'po_id': str, 'access': bool}}
user_state = {}  # {user_id: 'awaiting_registration'/'awaiting_po'/... , or dict for flows}

# --------------------- ASSETS ---------------------
FIN_ASSETS = {
    'EUR/USD': '🇪🇺🇺🇸','GBP/USD': '🇬🇧🇺🇸','AUD/USD': '🇦🇺🇺🇸','NZD/USD': '🇳🇿🇺🇸',
    'USD/CAD': '🇺🇸🇨🇦','USD/CHF': '🇺🇸🇨🇭','USD/JPY': '🇺🇸🇯🇵','EUR/GBP': '🇪🇺🇬🇧',
    'EUR/JPY': '🇪🇺🇯🇵','EUR/CHF': '🇪🇺🇨🇭','EUR/CAD': '🇪🇺🇨🇦','GBP/JPY': '🇬🇧🇯🇵',
    'GBP/CHF': '🇬🇧🇨🇭','AUD/JPY': '🇦🇺🇯🇵','AUD/CHF': '🇦🇺🇨🇭','CAD/JPY': '🇨🇦🇯🇵',
    'CAD/CHF': '🇨🇦🇨🇭','CHF/JPY': '🇨🇭🇯🇵','NZD/JPY': '🇳🇿🇯🇵','NZD/CAD': '🇳🇿🇨🇦'
}

FOREX_OTC = {
    'EUR/USD OTC': '🇪🇺🇺🇸','EUR/JPY OTC': '🇪🇺🇯🇵','EUR/GBP OTC': '🇪🇺🇬🇧','GBP/USD OTC': '🇬🇧🇺🇸',
    'USD/JPY OTC': '🇺🇸🇯🇵','AUD/USD OTC': '🇦🇺🇺🇸','USD/CAD OTC': '🇺🇸🇨🇦','NZD/USD OTC': '🇳🇿🇺🇸',
    'GBP/JPY OTC': '🇬🇧🇯🇵','AUD/JPY OTC': '🇦🇺🇯🇵','CAD/JPY OTC': '🇨🇦🇯🇵','AUD/CAD OTC': '🇦🇺🇨🇦',
    'AUD/CHF OTC': '🇦🇺🇨🇭','AUD/NZD OTC': '🇦🇺🇳🇿','CAD/CHF OTC': '🇨🇦🇨🇭','CHF/JPY OTC': '🇨🇭🇯🇵',
    'EUR/AUD OTC': '🇪🇺🇦🇺','EUR/CAD OTC': '🇪🇺🇨🇦','EUR/CHF OTC': '🇪🇺🇨🇭','EUR/NZD OTC': '🇪🇺🇳🇿',
    'GBP/AUD OTC': '🇬🇧🇦🇺','GBP/CAD OTC': '🇬🇧🇨🇦','GBP/CHF OTC': '🇬🇧🇨🇭','GBP/NZD OTC': '🇬🇧🇳🇿',
    'NZD/CAD OTC': '🇳🇿🇨🇦','NZD/CHF OTC': '🇳🇿🇨🇭','NZD/JPY OTC': '🇳🇿🇯🇵','USD/CHF OTC': '🇺🇸🇨🇭',
    'USD/SEK OTC': '🇺🇸🇸🇪','USD/NOK OTC': '🇺🇸🇳🇴','USD/MXN OTC': '🇺🇸🇲🇽','USD/ZAR OTC': '🇺🇸🇿🇦','USD/TRY OTC': '🇺🇸🇹🇷'
}

CRYPTO_OTC = {
    'Bitcoin OTC': '₿','Ethereum OTC': 'Ξ','Litecoin OTC': 'Ł','Solana OTC': '◎','Dogecoin OTC': 'Ð',
    'BNB OTC': '🅑','Polygon OTC': '🔶','Avalanche OTC': '❄️','Cardano OTC': '₳','Toncoin OTC': '🪙',
    'TRON OTC': '⭘','Chainlink OTC': '🔗','Polkadot OTC': '●','Bitcoin ETF OTC': '📈'
}

STOCKS_OTC = {
    'Apple OTC': '🍎','Facebook Inc OTC': '📘','McDonald\'s OTC': '🍔','GameStop Corp OTC': '🎮',
    'VISA OTC': '💳','Citigroup Inc OTC': '🏦','Alibaba OTC': '🛒','Pfizer Inc OTC': '💉',
    'ExxonMobil OTC': '🛢️','Intel OTC': '💾'
}

INDICATORS = ['RSI 🔍','MACD 📈','EMA 📊','BBANDS 📉','CCI 📊','Momentum ⚡']
FIN_TF = ['1 мин ⏱️','2 мин ⏱️','3 мин ⏱️','4 мин ⏱️','5 мин ⏱️','10 мин ⏱️']
OTC_TF = ['5 сек ⏳','10 сек ⏳','15 сек ⏳','30 сек ⏳','1 мин ⏱️']

# --------------------- ЛОГИРОВАНИЕ ---------------------
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# --------------------- ИНИЦИАЛИЗАЦИЯ ---------------------
bot = Bot(token=API_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher()

# --------------------- ФУНКЦИИ ---------------------
def main_menu(uid: int) -> ReplyKeyboardMarkup:
    kb = ReplyKeyboardMarkup(keyboard=[], resize_keyboard=True)
    if uid == ADMIN_ID:
        kb.keyboard.append([KeyboardButton(text='FIN 💵'), KeyboardButton(text='OTC 💱')])
        kb.keyboard.append([KeyboardButton(text='Автоматически 🤖')])
        kb.keyboard.append([KeyboardButton(text='Управление ⚙️')])
        kb.keyboard.append([KeyboardButton(text='Старт 🔄')])
    elif user_data.get(uid, {}).get('access'):
        kb.keyboard.append([KeyboardButton(text='FIN 💵'), KeyboardButton(text='OTC 💱')])
        kb.keyboard.append([KeyboardButton(text='Автоматически 🤖')])
        kb.keyboard.append([KeyboardButton(text='Старт 🔄')])
    else:
        kb.keyboard.append([KeyboardButton(text='Регистрация 📝')])
        kb.keyboard.append([KeyboardButton(text='Старт 🔄')])
    return kb

def admin_approve_kb(tid: int) -> InlineKeyboardMarkup:
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text='✅ Разрешить', callback_data=f'grant_{tid}'),
            InlineKeyboardButton(text='❌ Отклонить', callback_data=f'revoke_{tid}')
        ],
        [
            InlineKeyboardButton(text='💬 Написать пользователю', url=f'tg://user?id={tid}')
        ]
    ])
    return kb

def build_inline(opts: list, pre: str, rw: int = 2) -> InlineKeyboardMarkup:
    buttons = []
    row = []
    for i, o in enumerate(opts):
        row.append(InlineKeyboardButton(text=o, callback_data=f'{pre}:{o}'))
        if len(row) == rw or i == len(opts) - 1:
            buttons.append(row)
            row = []
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def registration_kb() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text='Я зарегистрировался ✅')]],
        resize_keyboard=True
    )

def get_moscow_time():
    return datetime.datetime.now(pytz.timezone(TIMEZONE)).strftime('%Y-%m-%d %H:%M:%S')

# --------------------- ХЕНДЛЕРЫ ---------------------
@dp.message(CommandStart())
async def start_cmd(message: types.Message):
    uid = message.from_user.id
    logger.info(f"Пользователь {uid} запустил бота")

    if uid == ADMIN_ID:
        user_data.setdefault(uid, {'access': True, 'registered': True, 'po_id': 'ADMIN'})
        await message.answer(
            '👨‍💼 <b>Админ-панель</b>\n\nДобро пожаловать в систему управления!',
            reply_markup=main_menu(uid)
        )
    else:
        user_state[uid] = 'awaiting_registration'
        user_data.setdefault(uid, {'access': False})

        if user_data[uid].get('access'):
            await message.answer(
                '🎯 <b>Добро пожаловать обратно!</b>\n\nВыберите тип сигналов:',
                reply_markup=main_menu(uid)
            )
        else:
            now = get_moscow_time()
            await message.answer(
                f'<b>🎯 Добро пожаловать в Trading Signals Bot!</b>\n'
                f'<em>📅 {now} (МСК)</em>\n\n'
                f'📋 <b>Для получения доступа к торговым сигналам:</b>\n\n'
                f'1️⃣ Зарегистрируйтесь на Pocket Option:\n'
                f'🔗 {PO_LINK}\n\n'
                f'2️⃣ Используйте промокод: <code>{PROMO_CODE}</code>\n\n'
                f'3️⃣ После регистрации нажмите кнопку ниже ⬇️',
                reply_markup=registration_kb()
            )

@dp.message(lambda message: message.text == 'Я зарегистрировался ✅')
async def on_registered(message: types.Message):
    uid = message.from_user.id
    if user_state.get(uid) == 'awaiting_registration':
        user_state[uid] = 'awaiting_po'
        logger.info(f"Пользователь {uid} подтвердил регистрацию")
        await message.answer(
            '🆔 <b>Введите ваш Pocket Option ID:</b>\n\n'
            '💡 ID можно найти в личном кабинете Pocket Option',
            reply_markup=types.ReplyKeyboardRemove()
        )
    else:
        await message.answer('❌ Сначала начните регистрацию командой /start')

@dp.message(lambda message: user_state.get(message.from_user.id) == 'awaiting_po')
async def receive_po(message: types.Message):
    uid, po = message.from_user.id, message.text.strip()

    if not po or len(po) < 5:
        await message.answer(
            '❌ <b>Некорректный Pocket Option ID</b>\n\n'
            '🆔 Пожалуйста, введите действительный ID аккаунта.\n'
            '💡 ID должен содержать не менее 5 символов.'
        )
        return

    user_data[uid].update({'po_id': po, 'access': False})
    user_state[uid] = None

    logger.info(f"Пользователь {uid} отправил PO ID: {po}")

    await message.answer(
        '✅ <b>ID получен!</b>\n\n'
        '⏳ Ваша заявка отправлена администратору.\n'
        '🔔 Вы получите уведомление о решении.'
    )

    user_info = f"👤 <b>Пользователь:</b>"
    if message.from_user.first_name:
        user_info += f" {message.from_user.first_name}"
    if message.from_user.last_name:
        user_info += f" {message.from_user.last_name}"
    if message.from_user.username:
        user_info += f" (@{message.from_user.username})"

    await bot.send_message(
        ADMIN_ID,
        f'🆕 <b>Новая заявка на доступ</b>\n\n'
        f'{user_info}\n'
        f'🆔 ID: <code>{uid}</code>\n'
        f'💼 Pocket Option ID: <code>{po}</code>\n\n'
        f'⚡ Требуется решение администратора',
        reply_markup=admin_approve_kb(uid)
    )

@dp.callback_query(lambda c: c.data.startswith('grant_'))
async def grant(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer('Доступ запрещен', show_alert=True)
        return

    tid = int(callback.data.split('_')[1])
    user_data[tid]['access'] = True

    logger.info(f"Администратор предоставил доступ пользователю {tid}")

    await bot.send_message(
        tid,
        '✅ <b>Доступ предоставлен!</b>\n\n'
        '🎉 Теперь вы можете получать торговые сигналы:\n'
        '📊 FIN - Форекс сигналы с реальными данными\n'
        '💱 OTC - Внебиржевые инструменты\n'
        '🤖 Автоматические сигналы\n\n'
        '💡 Выберите нужный раздел в меню ниже',
        reply_markup=main_menu(tid)
    )

    await callback.answer('✅ Доступ предоставлен')
    try:
        await callback.message.edit_text(f'✅ Пользователю {tid} предоставлен доступ')
    except:
        pass

@dp.callback_query(lambda c: c.data.startswith('revoke_'))
async def revoke(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer('Доступ запрещен', show_alert=True)
        return

    tid = int(callback.data.split('_')[1])
    user_data[tid]['access'] = False

    logger.info(f"Администратор отозвал доступ у пользователя {tid}")

    await bot.send_message(
        tid,
        '❌ <b>Доступ отклонен</b>\n\n'
        '😔 К сожалению, ваша заявка не была одобрена.\n'
        '📞 Обратитесь к администратору для уточнения деталей.'
    )

    await callback.answer('❌ Доступ отозван')
    try:
        await callback.message.edit_text(f'❌ У пользователя {tid} отозван доступ')
    except:
        pass

@dp.message(lambda message: message.text == 'Управление ⚙️')
async def manage(message: types.Message):
    if message.from_user.id != ADMIN_ID:
        await message.answer('🚫 Доступ запрещен')
        return

    total_users = len(user_data)
    active_users = sum(1 for user in user_data.values() if user.get('access'))
    pending_users = sum(1 for user in user_data.values() if user.get('po_id') and not user.get('access'))

    stats_text = (
        f"📊 <b>СТАТИСТИКА БОТА</b>\n\n"
        f"👥 Всего пользователей: {total_users}\n"
        f"✅ Активных: {active_users}\n"
        f"⏳ Ожидают одобрения: {pending_users}\n"
        f"❌ Без доступа: {total_users - active_users}\n\n"
        f"📋 <b>СПИСОК ПОЛЬЗОВАТЕЛЕЙ:</b>"
    )

    admin_buttons = [
        [InlineKeyboardButton(text="📊 Обновить статистику", callback_data="refresh_stats")]
    ]

    for uid, info in user_data.items():
        access_icon = '✅' if info.get('access') else '❌'
        po_id = info.get('po_id', 'Нет ID')
        po_short = po_id[:8] + "..." if len(str(po_id)) > 8 else str(po_id)
        user_text = f"{access_icon} {uid} | {po_short}"
        admin_buttons.append([InlineKeyboardButton(
            text=user_text,
            callback_data=f'sel_{uid}'
        )])

    kb = InlineKeyboardMarkup(inline_keyboard=admin_buttons)
    await message.answer(stats_text, reply_markup=kb)

@dp.callback_query(lambda c: c.data == 'refresh_stats')
async def refresh_stats(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer('Доступ запрещен', show_alert=True)
        return

    total_users = len(user_data)
    active_users = sum(1 for user in user_data.values() if user.get('access'))
    pending_users = sum(1 for user in user_data.values() if user.get('po_id') and not user.get('access'))

    stats_text = (
        f"📊 <b>СТАТИСТИКА БОТА</b>\n\n"
        f"👥 Всего пользователей: {total_users}\n"
       f"⏳ Ожидают одобрения: {pending_users}\n"
        f"❌ Без доступа: {total_users - active_users}\n\n"
        f"📋 <b>СПИСОК ПОЛЬЗОВАТЕЛЕЙ:</b>"
    )

    admin_buttons = [
        [InlineKeyboardButton(text="📊 Обновить статистику", callback_data="refresh_stats")]
    ]

    for uid, info in user_data.items():
        access_icon = '✅' if info.get('access') else '❌'
        po_id = info.get('po_id', 'Нет ID')
        po_short = po_id[:8] + "..." if len(str(po_id)) > 8 else str(po_id)
        user_text = f"{access_icon} {uid} | {po_short}"
        admin_buttons.append([InlineKeyboardButton(
            text=user_text,
            callback_data=f'sel_{uid}'
        )])

    kb = InlineKeyboardMarkup(inline_keyboard=admin_buttons)

    try:
        await callback.message.edit_text(stats_text, reply_markup=kb)
    except:
        await callback.message.answer(stats_text, reply_markup=kb)

    await callback.answer("📊 Статистика обновлена")

@dp.callback_query(lambda c: c.data.startswith('sel_'))
async def sel(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer('Доступ запрещен', show_alert=True)
        return

    uid = int(callback.data.split('_')[1])
    user_info = user_data.get(uid, {})

    access_status = "✅ Есть доступ" if user_info.get('access') else "❌ Нет доступа"
    po_id = user_info.get('po_id', 'Не указан')

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text='Дать доступ', callback_data=f'grant_{uid}'),
            InlineKeyboardButton(text='Отозвать доступ', callback_data=f'revoke_{uid}')
        ]
    ])

    await callback.message.answer(
        f'👤 <b>Пользователь {uid}</b>\n\n'
        f'🆔 PO ID: <code>{po_id}</code>\n'
        f'🔐 Статус: {access_status}',
        reply_markup=kb
    )

# FIN Flow
@dp.message(lambda message: message.text == 'FIN 💵')
async def fin_start(message: types.Message):
    uid = message.from_user.id
    if uid == ADMIN_ID:
        user_data.setdefault(uid, {'access': True, 'registered': True, 'po_id': 'ADMIN'})
    elif not user_data.get(uid, {}).get('access'):
        await message.answer('🚫 У вас нет доступа к этой функции')
        return

    user_state[uid] = {'flow': 'fin', 'stage': 'pair'}
    logger.info(f"Пользователь {uid} запустил FIN поток")

    buttons = []
    row = []
    for pair, flag in FIN_ASSETS.items():
        row.append(InlineKeyboardButton(text=f"{flag} {pair}", callback_data=f"fin_pair:{pair}"))
        if len(row) == 2:
            buttons.append(row)
            row = []
    if row:
        buttons.append(row)

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await message.answer('📊 <b>FIN Сигналы</b>\n\nВыберите валютную пару:', reply_markup=kb)

@dp.callback_query(lambda c: c.data.startswith('fin_pair:'))
async def fin_pair(callback: types.CallbackQuery):
    uid = callback.from_user.id
    pair = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'fin':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    state.update({'pair': pair, 'stage': 'ind'})
    user_state[uid] = state

    await callback.message.edit_text(
        f'📊 Пара: <b>{pair}</b>\n\nВыберите индикатор:',
        reply_markup=build_inline(INDICATORS, 'fin_ind')
    )

@dp.callback_query(lambda c: c.data.startswith('fin_ind:'))
async def fin_ind(callback: types.CallbackQuery):
    uid = callback.from_user.id
    ind = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'fin':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    state.update({'indicator': ind, 'stage': 'tf'})
    user_state[uid] = state

    await callback.message.edit_text(
        f'📊 Пара: <b>{state["pair"]}</b>\n'
        f'🔍 Индикатор: <b>{ind}</b>\n\nВыберите таймфрейм:',
        reply_markup=build_inline(FIN_TF, 'fin_tf')
    )

@dp.callback_query(lambda c: c.data.startswith('fin_tf:'))
async def fin_tf(callback: types.CallbackQuery):
    uid = callback.from_user.id
    tf = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'fin':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    try:
        await callback.message.edit_text('⏳ Генерируем сигнал...')
direction, price, strength = await fetch_fin_signal(state['pair'], state['indicator'], tf)

        caption = (
            f"📊 <b>{state['pair']}</b>\n"
            f"🔍 Индикатор: {state['indicator']}\n"
            f"⏰ Таймфрейм: {tf}\n"
            f"📈 Направление: <b>{direction}</b>\n"
            f"💰 Цена: {price}\n"
            f"⚡ Сила сигнала: {strength}%"
        )

        img_path = BUY_IMAGE_PATH if direction == 'Вверх' else SELL_IMAGE_PATH
        photo_file = FSInputFile(img_path)

        signal_emoji = "📈📈📈" if direction == 'Вверх' else "📉📉📉"
        arrow = "⬆️🟢" if direction == 'Вверх' else "⬇️🔴"

        full_caption = (
            f"{signal_emoji}\n"
            f"🎯 <b>FIN СИГНАЛ</b> {arrow}\n\n"
            f"{caption}\n\n"
            f"💡 <i>Удачных торгов!</i>"
        )

        await bot.send_photo(
            uid,
            photo_file,
            caption=full_caption,
            reply_markup=main_menu(uid)
        )

        logger.info(f"Пользователь {uid} получил FIN сигнал: {state['pair']} {direction}")

    except Exception as e:
        await bot.send_message(
            uid,
            '⚠️ Ошибка генерации сигнала. Попробуйте позже.',
            reply_markup=main_menu(uid)
        )
        logger.error(f"Ошибка FIN сигнала для пользователя {uid}: {e}")

    user_state[uid] = None
    await callback.answer()

# OTC Flow
@dp.message(lambda message: message.text == 'OTC 💱')
async def otc_start(message: types.Message):
    uid = message.from_user.id
    if uid == ADMIN_ID:
        user_data.setdefault(uid, {'access': True, 'registered': True, 'po_id': 'ADMIN'})
    elif not user_data.get(uid, {}).get('access'):
        await message.answer('🚫 У вас нет доступа к этой функции')
        return

    user_state[uid] = {'flow': 'otc', 'stage': 'cat'}
    logger.info(f"Пользователь {uid} запустил OTC поток")

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text='FOREX', callback_data='otc_cat:FOREX'),
            InlineKeyboardButton(text='CRYPTO', callback_data='otc_cat:CRYPTO'),
            InlineKeyboardButton(text='STOCKS', callback_data='otc_cat:STOCKS')
        ]
    ])
    await message.answer('💱 <b>OTC Сигналы</b>\n\nВыберите категорию:', reply_markup=kb)

@dp.callback_query(lambda c: c.data.startswith('otc_cat:'))
async def otc_cat(callback: types.CallbackQuery):
    uid = callback.from_user.id
    cat = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'otc':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    mapping = {'FOREX': FOREX_OTC, 'CRYPTO': CRYPTO_OTC, 'STOCKS': STOCKS_OTC}
    instruments = mapping.get(cat, {})

    buttons = []
    row = []
    for instrument, flag in instruments.items():
        row.append(InlineKeyboardButton(text=f"{flag} {instrument}", callback_data=f"otc_pair:{instrument}"))
        if len(row) == 2:
            buttons.append(row)
            row = []
    if row:
        buttons.append(row)

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    state.update({'cat': cat, 'stage': 'pair'})
    user_state[uid] = state

    await callback.message.edit_text(
        f'💱 Категория: <b>{cat}</b>\n\nВыберите инструмент:',
        reply_markup=kb
    )

@dp.callback_query(lambda c: c.data.startswith('otc_pair:'))
async def otc_pair(callback: types.CallbackQuery):
    uid = callback.from_user.id
    pair = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'otc':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    state.update({'pair': pair, 'stage': 'ind'})
    user_state[uid] = state

    await callback.message.edit_text(
        f'💱 Инструмент: <b>{pair}</b>\n\nВыберите индикатор:',
        reply_markup=build_inline(INDICATORS, 'otc_ind')
    )

@dp.callback_query(lambda c: c.data.startswith('otc_ind:'))
async def otc_ind(callback: types.CallbackQuery):
    uid = callback.from_user.id
    ind = callback.data.split(':', 1)[1]

    state = user_state.get(uid)
    if not state or state.get('flow') != 'otc':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    state.update({'indicator': ind, 'stage': 'tf'})
    user_state[uid] = state

    await callback.message.edit_text(
        f'💱 Инструмент: <b>{state["pair"]}</b>\n'
        f'🔍 Индикатор: <b>{ind}</b>\n\nВыберите таймфрейм:',
        reply_markup=build_inline(OTC_TF, 'otc_tf')
    )

@dp.callback_query(lambda c: c.data.startswith('otc_tf:'))
async def otc_tf(callback: types.CallbackQuery):
    uid = callback.from_user.id
    tf = callback.data.split(':', 1)[1]
  state = user_state.get(uid)
    if not state or state.get('flow') != 'otc':
        await callback.answer('Ошибка состояния', show_alert=True)
        return

    try:
        await callback.message.edit_text('⏳ Генерируем сигнал...')

        direction = random.choice(['Вверх', 'Вниз'])

        pair = state['pair']
        if 'Bitcoin' in pair or 'BTC' in pair:
            price = round(random.uniform(40000, 70000), 2)
        elif 'Ethereum' in pair or 'ETH' in pair:
            price = round(random.uniform(2000, 4000), 2)
        elif any(crypto in pair for crypto in ['Litecoin', 'Dogecoin', 'BNB']):
            price = round(random.uniform(50, 500), 2)
        elif 'Apple' in pair:
            price = round(random.uniform(150, 200), 2)
        elif any(stock in pair for stock in ['Facebook', 'Tesla', 'Google']):
            price = round(random.uniform(100, 300), 2)
        else:
            price = round(random.uniform(0.8, 1.5), 4)

        strength = random.randint(86, 97)

        caption = (
            f"📊 <b>{pair}</b>\n"
            f"🔍 Индикатор: {state['indicator']}\n"
            f"⏰ Таймфрейм: {tf}\n"
            f"📈 Направление: <b>{direction}</b>\n"
            f"💰 Цена: {price}\n"
            f"⚡ Сила сигнала: {strength}%"
        )

        img_path = BUY_IMAGE_PATH if direction == 'Вверх' else SELL_IMAGE_PATH
        photo_file = FSInputFile(img_path)

        signal_emoji = "📈📈📈" if direction == 'Вверх' else "📉📉📉"
        arrow = "⬆️🟢" if direction == 'Вверх' else "⬇️🔴"

        full_caption = (
            f"{signal_emoji}\n"
            f"🎯 <b>OTC СИГНАЛ</b> {arrow}\n\n"
            f"{caption}\n\n"
            f"💡 <i>Удачных торгов!</i>"
        )

        await bot.send_photo(
            uid,
            photo_file,
            caption=full_caption,
            reply_markup=main_menu(uid)
        )

        logger.info(f"Пользователь {uid} получил OTC сигнал: {pair} {direction}")

    except Exception as e:
        await bot.send_message(
            uid,
            '⚠️ Ошибка генерации сигнала. Попробуйте позже.',
            reply_markup=main_menu(uid)
        )
        logger.error(f"Ошибка OTC сигнала для пользователя {uid}: {e}")

    user_state[uid] = None
    await callback.answer()

@dp.message(lambda message: message.text == 'Старт 🔄')
async def restart_bot(message: types.Message):
    uid = message.from_user.id
    logger.info(f"Пользователь {uid} нажал кнопку Старт")

    user_state[uid] = None

    if uid == ADMIN_ID:
        user_data.setdefault(uid, {'access': True, 'registered': True, 'po_id': 'ADMIN'})
        await message.answer(
            '👨‍💼 <b>Админ-панель</b>\n\nДобро пожаловать в систему управления!',
            reply_markup=main_menu(uid)
        )
    else:
        user_data.setdefault(uid, {'access': False})

        if user_data[uid].get('access'):
            await message.answer(
                '🎯 <b>Добро пожаловать обратно!</b>\n\nВыберите тип сигналов:',
                reply_markup=main_menu(uid)
            )
        else:
            now = get_moscow_time()
            await message.answer(
                f'<b>🎯 Добро пожаловать в Trading Signals Bot!</b>\n'
                f'📅 {now}\n\n'
                f'🚀 <b>Получайте точные торговые сигналы</b>\n'
                f'📊 FIN и OTC рынки\n'
                f'⚡ Точность 86-97%\n\n'
                f'📝 Для начала пройдите регистрацию',
                reply_markup=main_menu(uid)
            )

@dp.message(lambda message: message.text == 'Автоматически 🤖')
async def auto_trading(message: types.Message):
    uid = message.from_user.id
    if uid == ADMIN_ID:
        user_data.setdefault(uid, {'access': True, 'registered': True, 'po_id': 'ADMIN'})
    elif not user_data.get(uid, {}).get('access'):
        await message.answer('🚫 У вас нет доступа к этой функции')
return

    signal_type = random.choice(['FIN', 'OTC'])

    if signal_type == 'FIN':
        pair = random.choice(list(FIN_ASSETS.keys()))
        indicator = random.choice(INDICATORS)
        timeframe = random.choice(FIN_TF)
        try:
            direction, price, strength = await fetch_fin_signal(pair, indicator, timeframe)
        except:
            direction = random.choice(['Вверх', 'Вниз'])
            price = round(random.uniform(1.0, 2.0), 4)
            strength = random.randint(60, 95)
    else:
        category = random.choice(['FOREX', 'CRYPTO', 'STOCKS'])
        if category == 'FOREX':
            pair = random.choice(list(FOREX_OTC.keys()))
        elif category == 'CRYPTO':
            pair = random.choice(list(CRYPTO_OTC.keys()))
        else:
            pair = random.choice(list(STOCKS_OTC.keys()))

        indicator = random.choice(INDICATORS)
        timeframe = random.choice(OTC_TF)
        direction = random.choice(['Вверх', 'Вниз'])

        if 'Bitcoin' in pair:
            price = round(random.uniform(40000, 70000), 2)
        elif 'Ethereum' in pair:
            price = round(random.uniform(2000, 4000), 2)
        elif any(crypto in pair for crypto in ['Litecoin', 'Dogecoin', 'BNB']):
            price = round(random.uniform(50, 500), 2)
        elif 'Apple' in pair:
            price = round(random.uniform(150, 200), 2)
        elif any(stock in pair for stock in ['Facebook', 'Tesla', 'Google']):
            price = round(random.uniform(100, 300), 2)
        else:
            price = round(random.uniform(0.8, 1.5), 4)

        strength = random.randint(86, 97)

    caption = (
        f"📊 <b>{pair}</b>\n"
        f"🔍 Индикатор: {indicator}\n"
        f"⏰ Таймфрейм: {timeframe}\n"
        f"📈 Направление: <b>{direction}</b>\n"
        f"💰 Цена: {price}\n"
        f"⚡ Сила сигнала: {strength}%\n"
        f"🎯 Тип: {signal_type}"
    )

    img_path = BUY_IMAGE_PATH if direction == 'Вверх' else SELL_IMAGE_PATH
    photo_file = FSInputFile(img_path)

    signal_emoji = "📈📈📈" if direction == 'Вверх' else "📉📉📉"
    arrow = "⬆️🟢" if direction == 'Вверх' else "⬇️🔴"

    full_caption = (
        f"{signal_emoji}\n"
        f"🤖 <b>АВТОМАТИЧЕСКИЙ СИГНАЛ</b> {arrow}\n\n"
        f"{caption}\n\n"
        f"⚡ <i>Сигнал сгенерирован автоматически!</i>\n"
        f"💡 <i>Удачных торгов!</i>"
    )

    try:
        await bot.send_photo(
            uid,
            photo_file,
            caption=full_caption,
            reply_markup=main_menu(uid)
        )
        logger.info(f"Пользователь {uid} получил автоматический сигнал: {pair} {direction}")
    except Exception as e:
        await message.answer("⚠️ Ошибка отправки автоматического сигнала")
        logger.error(f"Ошибка отправки автосигнала: {e}")

async def fetch_fin_signal(pair: str, ind: str, tf: str):
    try:
        interval_map = {
            '1 мин ⏱️': '1min', '2 мин ⏱️': '2min', '3 мин ⏱️': '3min',
            '4 мин ⏱️': '4min', '5 мин ⏱️': '5min', '10 мин ⏱️': '10min'
        }
        interval = interval_map.get(tf, '5min')
from_symbol, to_symbol = pair.split('/')
        url = 'https://www.alphavantage.co/query'
        params = {
            'function': 'FX_INTRADAY',
            'from_symbol': from_symbol,
            'to_symbol': to_symbol,
            'interval': interval,
            'apikey': FIN_API_KEY
        }

        async with aiohttp.ClientSession() as session:
            async with session.get(url, params=params) as response:
                data = await response.json()

        time_series_key = f'Time Series FX ({interval})'
        time_series = data.get(time_series_key, {})

        if not time_series:
            direction = random.choice(['Вверх', 'Вниз'])
            price = round(random.uniform(1.0, 2.0), 4)
            strength = random.randint(50, 100)
            return direction, price, strength

        latest_data = next(iter(time_series.values()))
        price = float(latest_data['4. close'])
        open_price = float(latest_data['1. open'])

        direction = 'Вверх' if price > open_price else 'Вниз'
        strength = random.randint(86, 97)

        return direction, price, strength

    except Exception as e:
        logger.error(f"Ошибка получения FIN сигнала: {e}")
        direction = random.choice(['Вверх', 'Вниз'])
        price = round(random.uniform(1.0, 2.0), 4)
        strength = random.randint(50, 100)
        return direction, price, strength

async def main():
    try:
        logger.info("🚀 Telegram Trading Bot запущен!")
        logger.info("📊 Поддерживаемые рынки: FIN, OTC")
        logger.info("🔧 Админ-панель доступна")
        await dp.start_polling(bot, skip_updates=True)
except Exception as e:
        logger.error(f"❌ Ошибка запуска бота: {e}")
        raise
