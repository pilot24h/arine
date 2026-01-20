#!/usr/bin/env python3
import asyncio
import logging
import datetime
from typing import Dict, Any, Optional

from aiogram import Bot, Dispatcher
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart, Command
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton

# ========== НАСТРОЙКА ЛОГИРОВАНИЯ ==========
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)

logger = logging.getLogger("pill_bot")

# ========== КОНФИГУРАЦИЯ ==========
API_TOKEN = '8165334459:AAHlZkOCIMDHbcW29NN_hW6iBE47Y5_nt5Q'
ADMIN_USERNAME = "@kirilka_zz"  # Имя пользователя для отправки отчетов

# ========== НАСТРОЙКИ ВРЕМЕНИ ==========
# Время по умолчанию (Санкт-Петербург - UTC+3)
DEFAULT_TIME_OFFSET = 3
CITIES = {
    "Санкт-Петербург": {
        "offset": 3,
        "pill_hour": 16,
        "pill_minute": 30,
        "reminder_hour": 14  # За 2 часа до приема (16-2=14)
    },
    "Райчихинск": {
        "offset": 9,
        "pill_hour": 16,  # В 16:30 по местному времени Райчихинска
        "pill_minute": 30,
        "reminder_hour": 14  # За 2 часа до приема (16-2=14)
    }
}

# ========== ИНИЦИАЛИЗАЦИЯ ==========
bot = Bot(token=API_TOKEN)
dp = Dispatcher()

# ========== ХРАНИЛИЩЕ ДАННЫХ ==========
# Структура: {user_id: {'city': 'Санкт-Петербург', 'last_taken': datetime, 'can_report': True}}
user_states: Dict[int, Dict[str, Any]] = {}

# ========== ФУНКЦИИ ДЛЯ РАБОТЫ СО ВРЕМЕНЕМ ==========

def get_user_local_time(user_id: int) -> datetime.datetime:
    """Возвращает текущее время с учетом выбранного города пользователя"""
    utc_now = datetime.datetime.utcnow()
    city = get_user_city(user_id)
    offset = CITIES[city]["offset"]
    local_now = utc_now + datetime.timedelta(hours=offset)
    return local_now

def get_user_city(user_id: int) -> str:
    """Возвращает выбранный город пользователя"""
    return user_states.get(user_id, {}).get('city', 'Санкт-Петербург')

def get_user_time_offset(user_id: int) -> int:
    """Возвращает смещение времени для пользователя"""
    city = get_user_city(user_id)
    return CITIES[city]["offset"]

def get_user_pill_time(user_id: int) -> tuple[int, int]:
    """Возвращает час и минуту приема таблеток для пользователя"""
    city = get_user_city(user_id)
    return CITIES[city]["pill_hour"], CITIES[city]["pill_minute"]

def get_user_reminder_time(user_id: int) -> tuple[int, int]:
    """Возвращает час и минуту напоминания (за 2 часа) для пользователя"""
    city = get_user_city(user_id)
    return CITIES[city]["reminder_hour"], CITIES[city]["pill_minute"]

def format_local_time(user_id: int, dt: datetime.datetime = None, format_str: str = '%H:%M:%S') -> str:
    """Форматирует время с учетом города пользователя"""
    if dt is None:
        dt = get_user_local_time(user_id)
    return dt.strftime(format_str)

# ========== ТЕКСТЫ СООБЩЕНИЙ ==========

def get_welcome_text(user_id: int) -> str:
    """Генерирует приветственное сообщение с учетом города пользователя"""
    city = get_user_city(user_id)
    pill_hour, pill_minute = get_user_pill_time(user_id)
    reminder_hour, reminder_minute = get_user_reminder_time(user_id)

    return f"""
💊 *Вспоминаем о приеме таблеточек)*

Это как всегда твой Кирилл🥰
Я помогу тебе не забывать принимать таблеточки

*📅 Расписание:*
• Основное напоминание: {pill_hour:02d}:{pill_minute:02d} ({city})
• Предварительное напоминание: {reminder_hour:02d}:{reminder_minute:02d} (за 2 часа)

📋 *Доступные команды:*
/start - Перезапустить бота
/help - Помощь
"""

def get_reminder_text(user_id: int, is_preliminary: bool = False) -> str:
    """Генерирует текст напоминания с учетом города пользователя"""
    city = get_user_city(user_id)
    pill_hour, pill_minute = get_user_pill_time(user_id)

    if is_preliminary:
        return f"""
⏰ *ПРЕДВАРИТЕЛЬНОЕ НАПОМИНАНИЕ*

Солнышко, через 2 часа нужно будет принять таблеточки! ⏳
Заранее подготовься и не забудь 💖
*Время приема ({city}):* {pill_hour:02d}:{pill_minute:02d}
"""
    else:
        return f"""
⏰ *НАПОМИНАНИЕ О ПРИЕМЕ ТАБЛЕТОЧЕК*

Солнышко, пора пить таблеточки! 
Я знаю ты не любишь, но надо❤️
Дай немного позаботиться о тебе 😚
*Время приема ({city}):* {pill_hour:02d}:{pill_minute:02d}
"""

PILL_TAKEN_TEXT = """
❤️🥰💋 *Чудесно, ты умничка! Ещё одна таблетка выпита.*

Запись о приеме сохранена🤗.
"""

PILL_REFUSED_TEXT = """
❌ *Ты отказалась принимать таблеточки?
Арина...., ты чего ахахахахх.*

Пожалуйста, вернись и прими их💋.
"""

# ========== ФУНКЦИИ ДЛЯ СОЗДАНИЯ КНОПОК ==========

def create_main_keyboard(user_id: Optional[int] = None):
    """Создает главную клавиатуру с учетом состояния пользователя"""
    keyboard = [
        [InlineKeyboardButton(text="Время до приёма таблеточек", callback_data="time_to_pill")],
    ]

    # Проверяем, может ли пользователь нажимать кнопку отчета
    can_report = user_states.get(user_id, {}).get('can_report', True) if user_id else True
    if can_report:
        keyboard.append([InlineKeyboardButton(text="Отчёт о приёме таблетки", callback_data="pill_report")])
    else:
        # Если отчет уже сделан, показываем неактивную кнопку
        keyboard.append([InlineKeyboardButton(text="✅ Отчёт уже отправлен", callback_data="already_reported")])

    keyboard.append([InlineKeyboardButton(text="⚙️ Настройки", callback_data="settings")])
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def create_report_keyboard():
    """Создает клавиатуру для отчета"""
    keyboard = [
        [InlineKeyboardButton(text="Я выпила)", callback_data="pill_taken")],
        [InlineKeyboardButton(text="Нее, не хочу)", callback_data="pill_refused")],
        [InlineKeyboardButton(text="🏠 Главная страница", callback_data="main_page")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def create_back_to_pill_keyboard():
    keyboard = [
        [InlineKeyboardButton(text="Вернуться назад и быстренько выпить таблетку🤗🥰)", callback_data="back_to_pill")],
        [InlineKeyboardButton(text="🏠 Главная страница", callback_data="main_page")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def create_back_to_main_keyboard():
    keyboard = [
        [InlineKeyboardButton(text="🏠 Главная страница", callback_data="main_page")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def create_settings_keyboard(user_id: int):
    """Создает клавиатуру настроек с текущим городом"""
    current_city = get_user_city(user_id)

    keyboard = []

    # Кнопки выбора города
    if current_city == "Санкт-Петербург":
        pill_hour, pill_minute = CITIES["Санкт-Петербург"]["pill_hour"], CITIES["Санкт-Петербург"]["pill_minute"]
        reminder_hour = CITIES["Санкт-Петербург"]["reminder_hour"]
        keyboard.append([InlineKeyboardButton(text=f"✅ Санкт-Петербург ({pill_hour:02d}:{pill_minute:02d})", callback_data="city_spb")])
        pill_hour, pill_minute = CITIES["Райчихинск"]["pill_hour"], CITIES["Райчихинск"]["pill_minute"]
        reminder_hour = CITIES["Райчихинск"]["reminder_hour"]
        keyboard.append([InlineKeyboardButton(text=f"Райчихинск ({pill_hour:02d}:{pill_minute:02d})", callback_data="city_raichikhinsk")])
    else:
        pill_hour, pill_minute = CITIES["Санкт-Петербург"]["pill_hour"], CITIES["Санкт-Петербург"]["pill_minute"]
        reminder_hour = CITIES["Санкт-Петербург"]["reminder_hour"]
        keyboard.append([InlineKeyboardButton(text=f"Санкт-Петербург ({pill_hour:02d}:{pill_minute:02d})", callback_data="city_spb")])
        pill_hour, pill_minute = CITIES["Райчихинск"]["pill_hour"], CITIES["Райчихинск"]["pill_minute"]
        reminder_hour = CITIES["Райчихинск"]["reminder_hour"]
        keyboard.append([InlineKeyboardButton(text=f"✅ Райчихинск ({pill_hour:02d}:{pill_minute:02d})", callback_data="city_raichikhinsk")])

    keyboard.append([InlineKeyboardButton(text="🏠 Главная страница", callback_data="main_page")])

    return InlineKeyboardMarkup(inline_keyboard=keyboard)

async def send_main_page(user_id: int):
    """Отправляет главную страницу"""
    welcome_text = (
        "💊 *Вспоминаем о приеме таблеточек)*\n\n"
        "Это как всегда твой Кирилл🥰\n"
        "Я помогу тебе не забывать принимать таблеточки"
    )
    
    await bot.send_message(
        chat_id=user_id,
        text=welcome_text,
        parse_mode=ParseMode.MARKDOWN,
        reply_markup=create_main_keyboard(user_id)
    )

# ========== ОБРАБОТЧИКИ КОМАНД ==========

@dp.message(CommandStart())
async def handle_start(message: Message):
    """Обработчик команды /start - перезапускает бота"""
    try:
        user_id = message.from_user.id
        username = message.from_user.first_name or "Пользователь"

        # Сохраняем текущие настройки города, если пользователь уже был зарегистрирован
        current_city = user_states.get(user_id, {}).get('city', 'Санкт-Петербург')

        # Перезаписываем данные пользователя (перезапуск)
        user_states[user_id] = {
            'started': True,
            'username': username,
            'city': current_city,  # Сохраняем выбранный город
            'join_date': get_user_local_time(user_id).isoformat(),
            'can_report': True  # Сбрасываем возможность отправки отчета
        }

        city = get_user_city(user_id)
        pill_hour, pill_minute = get_user_pill_time(user_id)
        reminder_hour, reminder_minute = get_user_reminder_time(user_id)

        logger.info(f"Перезапуск бота для пользователя: {user_id} ({username}) - город: {city}")

        # ОДНО сообщение при перезапуске
        welcome_message = (
            f"🔄 *Бот перезапущен!*\n\n"
            f"Привет снова, {username}! 😊\n"
            f"Твои настройки сохранены:\n"
            f"• Город: {city}\n"
            f"• Основное время приема: {pill_hour:02d}:{pill_minute:02d}\n"
            f"• Предварительное напоминание: {reminder_hour:02d}:{reminder_minute:02d}\n"
            f"• Часовой пояс: UTC+{CITIES[city]['offset']}\n\n"
            f"Вот главное меню:"
        )

        await message.answer(
            welcome_message,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_main_keyboard(user_id)
        )

        # Отправляем приветственное сообщение
        await asyncio.sleep(0.5)
        await message.answer(
            get_welcome_text(user_id),
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_main_keyboard(user_id)
        )

    except Exception as e:
        logger.error(f"Ошибка в handle_start: {e}")
        await message.answer("Ошибка при перезапуске бота. Попробуйте снова.")

@dp.message(Command("help"))
async def handle_help(message: Message):
    """Обработчик команды /help"""
    help_text = (
        "📚 *Помощь по боту:*\n\n"
        "*Доступные команды:*\n"
        "/start - Перезапустить бота\n"
        "/help - Показать это сообщение\n\n"
        "*Основные функции:*\n"
        "• Два ежедневных напоминания:\n"
        "  - За 2 часа до приема\n"
        "  - Время приема таблеток\n"
        "• Отчет о приеме таблеток\n"
        "• Настройка времени по городам\n"
        "• Отслеживание времени до следующего приема\n\n"
        "*Настройки:*\n"
        "В настройках можно выбрать город:\n"
        "• Санкт-Петербург (UTC+3)\n"
        "• Райчихинск (UTC+9)\n\n"
        "*Отчеты:*\n"
        "Все отчеты о приеме таблеток отправляются @kirilka_zz\n\n"
        "Бот автоматически адаптирует время напоминаний под выбранный город!"
    )

    await message.answer(
        help_text,
        parse_mode=ParseMode.MARKDOWN,
        reply_markup=create_main_keyboard(message.from_user.id)
    )

# ========== ОБРАБОТЧИКИ КНОПОК ==========

@dp.callback_query(lambda c: c.data == "time_to_pill")
async def handle_time_to_pill(callback: CallbackQuery):
    try:
        await callback.answer()

        user_id = callback.from_user.id
        now = get_user_local_time(user_id)
        pill_hour, pill_minute = get_user_pill_time(user_id)
        reminder_hour, reminder_minute = get_user_reminder_time(user_id)
        city = get_user_city(user_id)

        # Время следующего приема
        pill_target_time = now.replace(hour=pill_hour, minute=pill_minute, second=0, microsecond=0)
        if now >= pill_target_time:
            pill_target_time += datetime.timedelta(days=1)

        # Время предварительного напоминания
        reminder_target_time = now.replace(hour=reminder_hour, minute=reminder_minute, second=0, microsecond=0)
        if now >= reminder_target_time:
            reminder_target_time += datetime.timedelta(days=1)

        # Время до приема
        time_to_pill = pill_target_time - now
        hours_to_pill = time_to_pill.seconds // 3600
        minutes_to_pill = (time_to_pill.seconds % 3600) // 60

        # Время до предварительного напоминания
        time_to_reminder = reminder_target_time - now
        hours_to_reminder = time_to_reminder.seconds // 3600
        minutes_to_reminder = (time_to_reminder.seconds % 3600) // 60

        # Форматирование времени
        def format_time(hours, minutes):
            if hours == 0:
                return f"{minutes} мин."
            else:
                return f"{hours} ч. {minutes} мин."

        response = (
            f"⏰💋 *Текущее время ({city}):* {now.strftime('%H:%M')}\n\n"
            f"📅🥰*Расписание:*\n"
            f"• Предварительное напоминание: {reminder_hour:02d}:{reminder_minute:02d}\n"
            f"• Основной прием: {pill_hour:02d}:{pill_minute:02d}\n\n"
            f"⏳❤️ *Время до событий:*\n"
            f"• До предварительного напоминания:\n {format_time(hours_to_reminder, minutes_to_reminder)}\n"
            f"• До приёма таблеточек: {format_time(hours_to_pill, minutes_to_pill)}"
        )

        await callback.message.answer(
            response,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_main_keyboard(user_id)
        )

    except Exception as e:
        logger.error(f"Ошибка в handle_time_to_pill: {e}")

@dp.callback_query(lambda c: c.data == "pill_report")
async def handle_pill_report(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id

        # Проверяем, можно ли еще отправлять отчет
        if not user_states.get(user_id, {}).get('can_report', True):
            await callback.answer("Вы уже отправили отчет сегодня!", show_alert=True)
            return

        await callback.answer()
        await callback.message.answer(
            "Выберите действие:",
            reply_markup=create_report_keyboard()
        )
    except Exception as e:
        logger.error(f"Ошибка в handle_pill_report: {e}")

@dp.callback_query(lambda c: c.data == "already_reported")
async def handle_already_reported(callback: CallbackQuery):
    """Обработчик нажатия на неактивную кнопку отчета"""
    await callback.answer("Вы уже отправили отчет сегодня!", show_alert=True)

@dp.callback_query(lambda c: c.data == "main_page")
async def handle_main_page(callback: CallbackQuery):
    """Обработчик кнопки 'Главная страница'"""
    try:
        user_id = callback.from_user.id
        await callback.answer("Возвращаемся на главную...")
        await send_main_page(user_id)
    except Exception as e:
        logger.error(f"Ошибка в handle_main_page: {e}")

async def send_report_to_admin(user_id: int, username: str, city: str, action: str, time: datetime.datetime):
    """Отправляет отчет администратору"""
    try:
        report_message = (
            f"📊 *ОТЧЕТ О ПРИЕМЕ ТАБЛЕТОК*\n\n"
            f"👤 *Пользователь:* {username}\n"
            f"🆔 *ID:* {user_id}\n"
            f"🏙️ *Город:* {city}\n"
            f"⏰ *Время:* {time.strftime('%H:%M:%S')}\n"
            f"📅 *Дата:* {time.strftime('%Y-%m-%d')}\n"
            f"✅ *Действие:* {action}"
        )

        # Отправляем отчет администратору
        await bot.send_message(
            chat_id=ADMIN_USERNAME,
            text=report_message,
            parse_mode=ParseMode.MARKDOWN
        )
        logger.info(f"Отчет отправлен администратору {ADMIN_USERNAME}")

    except Exception as e:
        logger.error(f"Ошибка при отправке отчета администратору: {e}")

@dp.callback_query(lambda c: c.data == "pill_taken")
async def handle_pill_taken(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id

        # Проверяем, можно ли еще отправлять отчет
        if not user_states.get(user_id, {}).get('can_report', True):
            await callback.answer("Вы уже отправили отчет сегодня!", show_alert=True)
            return

        await callback.answer("Отлично! 😚")

        now = get_user_local_time(user_id)
        city = get_user_city(user_id)
        username = user_states[user_id].get('username', 'Неизвестный пользователь')

        # Сохраняем время приема и блокируем повторные отчеты
        user_states[user_id]['last_taken'] = now.isoformat()
        user_states[user_id]['status'] = 'taken'
        user_states[user_id]['can_report'] = False

        # Отправляем отчет администратору
        await send_report_to_admin(
            user_id=user_id,
            username=username,
            city=city,
            action="Приняла таблетку ✅",
            time=now
        )

        response = PILL_TAKEN_TEXT + f"\n\n🕒 *Время приёма ({city}):* {now.strftime('%H:%M:%S')}"

        await callback.message.answer(
            response,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_back_to_main_keyboard()
        )

        logger.info(f"Пользователь {user_id} принял таблетку в {now} ({city})")

    except Exception as e:
        logger.error(f"Ошибка в handle_pill_taken: {e}")

@dp.callback_query(lambda c: c.data == "pill_refused")
async def handle_pill_refused(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id

        # Проверяем, можно ли еще отправлять отчет
        if not user_states.get(user_id, {}).get('can_report', True):
            await callback.answer("Вы уже отправили отчет сегодня!", show_alert=True)
            return

        await callback.answer()

        now = get_user_local_time(user_id)
        city = get_user_city(user_id)
        username = user_states[user_id].get('username', 'Неизвестный пользователь')

        user_states[user_id]['status'] = 'refused'
        user_states[user_id]['can_report'] = False

        # Отправляем отчет администратору об отказе
        await send_report_to_admin(
            user_id=user_id,
            username=username,
            city=city,
            action="Отказалась от приема таблетки ❌",
            time=now
        )

        await callback.message.answer(
            PILL_REFUSED_TEXT,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_back_to_pill_keyboard()
        )

        logger.warning(f"Пользователь {user_id} отказался от таблетки")

    except Exception as e:
        logger.error(f"Ошибка в handle_pill_refused: {e}")

@dp.callback_query(lambda c: c.data == "back_to_pill")
async def handle_back_to_pill(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id

        # Разблокируем возможность отчета при возврате
        user_states[user_id]['can_report'] = True

        await callback.answer()
        await callback.message.answer(
            "Пожалуйста, прими таблеточку) :",
            reply_markup=create_report_keyboard()
        )
    except Exception as e:
        logger.error(f"Ошибка в handle_back_to_pill: {e}")

@dp.callback_query(lambda c: c.data == "back_to_main")
async def handle_back_to_main(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id
        await callback.answer()
        await callback.message.answer(
            "Возвращаемся на главную...",
            reply_markup=create_main_keyboard(user_id)
        )
    except Exception as e:
        logger.error(f"Ошибка в handle_back_to_main: {e}")

@dp.callback_query(lambda c: c.data == "settings")
async def handle_settings(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id
        await callback.answer()

        city = get_user_city(user_id)
        pill_hour, pill_minute = get_user_pill_time(user_id)
        reminder_hour, reminder_minute = get_user_reminder_time(user_id)

        settings_text = (
            f"⚙️ *Настройки*\n\n"
            f"*Текущий город:* {city}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Предварительное напоминание:* {reminder_hour:02d}:{reminder_minute:02d}\n\n"
            f"Выберите город для смены времени:"
        )

        await callback.message.answer(
            settings_text,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_settings_keyboard(user_id)
        )
    except Exception as e:
        logger.error(f"Ошибка в handle_settings: {e}")

@dp.callback_query(lambda c: c.data == "city_spb")
async def handle_city_spb(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id
        user_states[user_id]['city'] = 'Санкт-Петербург'

        await callback.answer("Город изменен на Санкт-Петербург! 🏙️")

        pill_hour, pill_minute = get_user_pill_time(user_id)
        reminder_hour, reminder_minute = get_user_reminder_time(user_id)
        city = get_user_city(user_id)

        update_text = (
            f"✅ *Настройки обновлены!*\n\n"
            f"*Новый город:* {city}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Предварительное напоминание:* {reminder_hour:02d}:{reminder_minute:02d}"
        )

        await callback.message.answer(
            update_text,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_settings_keyboard(user_id)
        )

        logger.info(f"Пользователь {user_id} изменил город на Санкт-Петербург")

    except Exception as e:
        logger.error(f"Ошибка в handle_city_spb: {e}")

@dp.callback_query(lambda c: c.data == "city_raichikhinsk")
async def handle_city_raichikhinsk(callback: CallbackQuery):
    try:
        user_id = callback.from_user.id
        user_states[user_id]['city'] = 'Райчихинск'

        await callback.answer("Город изменен на Райчихинск! 🌄")

        pill_hour, pill_minute = get_user_pill_time(user_id)
        reminder_hour, reminder_minute = get_user_reminder_time(user_id)
        city = get_user_city(user_id)

        update_text = (
            f"✅ *Настройки обновлены!*\n\n"
            f"*Новый город:* {city}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Предварительное напоминание:* {reminder_hour:02d}:{reminder_minute:02d}"
        )

        await callback.message.answer(
            update_text,
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_settings_keyboard(user_id)
        )

        logger.info(f"Пользователь {user_id} изменил город на Райчихинск")

    except Exception as e:
        logger.error(f"Ошибка в handle_city_raichikhinsk: {e}")

# ========== ФУНКЦИЯ ДЛЯ ЕЖЕДНЕВНЫХ НАПОМИНАНИЙ ==========

async def check_daily_reminder():
    while True:
        try:
            utc_now = datetime.datetime.utcnow()

            for user_id, user_data in list(user_states.items()):
                try:
                    # Получаем время пользователя
                    city = get_user_city(user_id)
                    offset = CITIES[city]["offset"]
                    user_local_time = utc_now + datetime.timedelta(hours=offset)

                    # Получаем время приема для пользователя
                    pill_hour, pill_minute = get_user_pill_time(user_id)
                    reminder_hour, reminder_minute = get_user_reminder_time(user_id)

                    # Проверяем, наступило ли время предварительного напоминания (за 2 часа)
                    if user_local_time.hour == reminder_hour and user_local_time.minute == reminder_minute:
                        logger.info(f"⏰🔔 Предварительное напоминание {reminder_hour:02d}:{reminder_minute:02d} ({city})! Отправляю пользователю {user_id}")

                        try:
                            # Отправляем напоминание
                            await bot.send_message(
                                chat_id=user_id,
                                text=get_reminder_text(user_id, is_preliminary=True),
                                parse_mode=ParseMode.MARKDOWN,
                                reply_markup=create_main_keyboard(user_id)
                            )

                            logger.info(f"✅ Предварительное напоминание отправлено пользователю {user_id} в {user_local_time.strftime('%H:%M:%S')} ({city})")

                        except Exception as e:
                            error_msg = str(e)
                            logger.error(f"❌ Не удалось отправить предварительное напоминание пользователю {user_id}: {error_msg}")

                    # Проверяем, наступило ли время основного приема
                    if user_local_time.hour == pill_hour and user_local_time.minute == pill_minute:
                        logger.info(f"⏰🥹 Основное напоминание {pill_hour:02d}:{pill_minute:02d} ({city})! Отправляю пользователю {user_id}")

                        try:
                            # Отправляем напоминание
                            await bot.send_message(
                                chat_id=user_id,
                                text=get_reminder_text(user_id, is_preliminary=False),
                                parse_mode=ParseMode.MARKDOWN,
                                reply_markup=create_main_keyboard(user_id)
                            )

                            # Разблокируем возможность отчета на новый день
                            user_states[user_id]['can_report'] = True

                            logger.info(f"✅ Основное напоминание отправлено пользователю {user_id} в {user_local_time.strftime('%H:%M:%S')} ({city})")

                        except Exception as e:
                            error_msg = str(e)
                            logger.error(f"❌ Не удалось отправить основное напоминание пользователю {user_id}: {error_msg}")

                            # Если пользователь заблокировал бота, удаляем его из списка
                            if any(phrase in error_msg.lower() for phrase in ["chat not found", "user is deactivated", "bot was blocked", "forbidden"]):
                                logger.warning(f"🗑️ Удаляю пользователя {user_id} из списка (заблокировал бота)")
                                del user_states[user_id]

                except Exception as e:
                    logger.error(f"⚠️ Ошибка при обработке пользователя {user_id}: {e}")
                    continue

            # Проверяем каждые 30 секунд
            await asyncio.sleep(30)

        except Exception as e:
            logger.error(f"⚠️ Ошибка в функции напоминаний: {e}")
            await asyncio.sleep(60)

# ========== ЗАПУСК БОТА ==========

async def main():
    try:
        logger.info("🤖 Бот запускается...")

        # Выводим информацию о городах
        logger.info("🌍 Настройки городов:")
        for city, data in CITIES.items():
            logger.info(f"   {city}: UTC+{data['offset']}, прием в {data['pill_hour']:02d}:{data['pill_minute']:02d}, напоминание за 2 часа в {data['reminder_hour']:02d}:{data['pill_minute']:02d}")

        logger.info(f"📨 Отчеты отправляются: {ADMIN_USERNAME}")

        # Проверяем подключение к Telegram API
        try:
            me = await bot.get_me()
            logger.info(f"✅ Бот успешно подключен!")
            logger.info(f"   Имя бота: {me.first_name}")
            logger.info(f"   Username: @{me.username}")
            logger.info(f"   ID бота: {me.id}")
        except Exception as e:
            logger.error(f"❌ Ошибка подключения к Telegram API: {e}")
            return

        # Запускаем задачу для проверки времени и отправки напоминаний
        logger.info("🔧 Запускаю службу напоминаний...")
        reminder_task = asyncio.create_task(check_daily_reminder())

        # Запускаем опрос бота
        logger.info("📡 Запускаю опрос сообщений...")
        logger.info("=" * 50)
        logger.info("✅ Бот готов к работе! Отправьте /start в Telegram")
        logger.info("=" * 50)

        await dp.start_polling(bot, skip_updates=True)

    except KeyboardInterrupt:
        logger.info("\n🛑 Бот остановлен пользователем (Ctrl+C)")
    except Exception as e:
        logger.error(f"💥 Критическая ошибка: {e}")
        import traceback
        logger.error(f"🔍 Трассировка ошибки:\n{traceback.format_exc()}")
    finally:
        logger.info("🔄 Завершение работы бота...")
        await bot.session.close()
        logger.info("👋 Бот завершил работу")

if __name__ == "__main__":
    asyncio.run(main())