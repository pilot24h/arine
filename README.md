#!/usr/bin/env python3
import asyncio
import logging
import datetime
from typing import Dict, Any, Optional

from aiogram import Bot, Dispatcher
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart
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

# ========== НАСТРОЙКИ ВРЕМЕНИ ==========
# Время по умолчанию (Санкт-Петербург - UTC+3)
DEFAULT_TIME_OFFSET = 3
CITIES = {
    "Санкт-Петербург": {
        "offset": 3,
        "pill_hour": 16,
        "pill_minute": 30
    },
    "Райчихинск": {
        "offset": 9,
        "pill_hour": 16,  # В 16:30 по местному времени Райчихинска
        "pill_minute": 30
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

    return f"""
💊 *Вспоминаем о приеме таблеточек)*

Это как всегда твой Кирилл🥰
Я помогу тебе не забывать принимать таблеточки
Ежедневно в {pill_hour:02d}:{pill_minute:02d} по местному времени ({city}) я буду присылать напоминание
"""


def get_reminder_text(user_id: int) -> str:
    """Генерирует текст напоминания с учетом города пользователя"""
    city = get_user_city(user_id)
    pill_hour, pill_minute = get_user_pill_time(user_id)

    return f"""
⏰ *НАПОМИНАНИЕ О ПРИЕМЕ ТАБЛЕТОЧЕК*

Солнышко, пора пить таблеточки! 
Я знаю ты не любишь, но надо❤️
Дай немного позаботиться о тебе 😚
*Время приема ({city}):* {pill_hour:02d}:{pill_minute:02d}
"""


PILL_TAKEN_TEXT = """
❤️🥰💋*Чудесно, ты умничка! Ещё одна таблетка выпита.*

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
        [InlineKeyboardButton(text="Нее, не хочу)", callback_data="pill_refused")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)


def create_back_to_pill_keyboard():
    keyboard = [
        [InlineKeyboardButton(text="Вернуться назад и быстренько выпить таблетку🤗🥰)", callback_data="back_to_pill")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)


def create_back_to_main_keyboard():
    keyboard = [
        [InlineKeyboardButton(text="Вернуться на главную", callback_data="back_to_main")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=keyboard)


def create_settings_keyboard(user_id: int):
    """Создает клавиатуру настроек с текущим городом"""
    current_city = get_user_city(user_id)

    keyboard = []

    # Кнопки выбора города
    if current_city == "Санкт-Петербург":
        pill_hour, pill_minute = CITIES["Санкт-Петербург"]["pill_hour"], CITIES["Санкт-Петербург"]["pill_minute"]
        keyboard.append([InlineKeyboardButton(text=f"✅ Санкт-Петербург ({pill_hour:02d}:{pill_minute:02d})",
                                              callback_data="city_spb")])
        pill_hour, pill_minute = CITIES["Райчихинск"]["pill_hour"], CITIES["Райчихинск"]["pill_minute"]
        keyboard.append([InlineKeyboardButton(text=f"Райчихинск ({pill_hour:02d}:{pill_minute:02d})",
                                              callback_data="city_raichikhinsk")])
    else:
        pill_hour, pill_minute = CITIES["Санкт-Петербург"]["pill_hour"], CITIES["Санкт-Петербург"]["pill_minute"]
        keyboard.append([InlineKeyboardButton(text=f"Санкт-Петербург ({pill_hour:02d}:{pill_minute:02d})",
                                              callback_data="city_spb")])
        pill_hour, pill_minute = CITIES["Райчихинск"]["pill_hour"], CITIES["Райчихинск"]["pill_minute"]
        keyboard.append([InlineKeyboardButton(text=f"✅ Райчихинск ({pill_hour:02d}:{pill_minute:02d})",
                                              callback_data="city_raichikhinsk")])

    keyboard.append([InlineKeyboardButton(text="⬅️ Назад", callback_data="back_to_main")])

    return InlineKeyboardMarkup(inline_keyboard=keyboard)


# ========== ОБРАБОТЧИКИ КОМАНД ==========

@dp.message(CommandStart())
async def handle_start(message: Message):
    try:
        user_id = message.from_user.id
        username = message.from_user.first_name or "Пользователь"

        # Инициализируем пользователя с настройками по умолчанию
        if user_id not in user_states:
            user_states[user_id] = {
                'started': True,
                'username': username,
                'city': 'Санкт-Петербург',
                'join_date': get_user_local_time(user_id).isoformat(),
                'can_report': True
            }

        logger.info(f"Новый пользователь: {user_id} ({username}) - город: {get_user_city(user_id)}")

        await message.answer(
            get_welcome_text(user_id),
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=create_main_keyboard(user_id)
        )

    except Exception as e:
        logger.error(f"Ошибка в handle_start: {e}")
        await message.answer("Ошибка при запуске бота. Попробуйте снова.")


# ========== ОБРАБОТЧИКИ КНОПОК ==========

@dp.callback_query(lambda c: c.data == "time_to_pill")
async def handle_time_to_pill(callback: CallbackQuery):
    try:
        await callback.answer()

        user_id = callback.from_user.id
        now = get_user_local_time(user_id)
        pill_hour, pill_minute = get_user_pill_time(user_id)
        city = get_user_city(user_id)

        target_time = now.replace(hour=pill_hour, minute=pill_minute, second=0, microsecond=0)

        if now >= target_time:
            target_time += datetime.timedelta(days=1)

        time_left = target_time - now
        hours = time_left.seconds // 3600
        minutes = (time_left.seconds % 3600) // 60

        # Если меньше часа, показываем только минуты
        if hours == 0:
            time_text = f"{minutes} мин."
        else:
            time_text = f"{hours} ч. {minutes} мин."

        response = (
            f"⏰💋 *Текущее время ({city}):* {now.strftime('%H:%M')}\n"
            f"⏳❤️ *До приёма таблеточек:* {time_text}\n"
            f"💊🥰 *Следующий приём таблеточек ({city}):* {pill_hour:02d}:{pill_minute:02d}"
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

        # Сохраняем время приема и блокируем повторные отчеты
        user_states[user_id]['last_taken'] = now.isoformat()
        user_states[user_id]['status'] = 'taken'
        user_states[user_id]['can_report'] = False

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

        user_states[user_id]['status'] = 'refused'
        user_states[user_id]['can_report'] = False

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

        settings_text = (
            f"⚙️ *Настройки*\n\n"
            f"*Текущий город:* {city}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}\n\n"
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
        city = get_user_city(user_id)

        update_text = (
            f"✅ *Настройки обновлены!*\n\n"
            f"*Новый город:* {city}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}"
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
        city = get_user_city(user_id)

        update_text = (
            f"✅ *Настройки обновлены!*\n\n"
            f"*Новый город:* {city}\n"
            f"*Время приема таблеток:* {pill_hour:02d}:{pill_minute:02d}\n"
            f"*Часовой пояс:* UTC+{CITIES[city]['offset']}"
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

                    # Проверяем, наступило ли время приема для этого пользователя
                    if user_local_time.hour == pill_hour and user_local_time.minute == pill_minute:
                        logger.info(
                            f"⏰🥹 Время {pill_hour:02d}:{pill_minute:02d} ({city})! Отправляю напоминание пользователю {user_id}")

                        # Формируем текст напоминания
                        reminder_text = get_reminder_text(user_id)
                        current_date_str = user_local_time.strftime('%Y-%m-%d')

                        # Разблокируем возможность отчета на новый день
                        user_states[user_id]['can_report'] = True

                        try:
                            await bot.send_message(
                                chat_id=user_id,
                                text=reminder_text,
                                parse_mode=ParseMode.MARKDOWN,
                                reply_markup=create_main_keyboard(user_id)
                            )

                            logger.info(
                                f"✅ Напоминание отправлено пользователю {user_id} в {user_local_time.strftime('%H:%M:%S')} ({city})")

                        except Exception as e:
                            error_msg = str(e)
                            logger.error(f"❌ Не удалось отправить напоминание пользователю {user_id}: {error_msg}")

                            # Если пользователь заблокировал бота, удаляем его из списка
                            if any(phrase in error_msg.lower() for phrase in
                                   ["chat not found", "user is deactivated", "bot was blocked", "forbidden"]):
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
            logger.info(f"   {city}: UTC+{data['offset']}, прием в {data['pill_hour']:02d}:{data['pill_minute']:02d}")

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
