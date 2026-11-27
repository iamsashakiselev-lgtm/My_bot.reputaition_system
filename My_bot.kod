
import asyncio
import sqlite3
import time
import html
import re
from datetime import datetime, timedelta
from functools import wraps
from aiogram import Bot, Dispatcher, F, BaseMiddleware
from aiogram.filters import CommandStart, Command
from aiogram.types import Message, ChatPermissions

# ==========================================
# ВСТАВЬ СЮДА СВОЙ РАБОЧИЙ ТОКЕН:
TOKEN = "9033109172:AAFdmIVseBy_cjj_Q2Fk27R7v4ZUqZKa-MY"
# ==========================================

dp = Dispatcher()
sticker_tracker = {}

def admin_only(func):
    @wraps(func)
    async def wrapper(message: Message, *args, **kwargs):
        if message.chat.type in ('group', 'supergroup'):
            try:
                member = await message.bot.get_chat_member(message.chat.id, message.from_user.id)
                if member.status not in ['administrator', 'creator']:
                    await message.reply("<b>❌ Данная команда доступна только для администрации</b>", parse_mode="HTML")
                    return
            except Exception:
                await message.reply("<b>❌ Данная команда доступна только для администрации</b>", parse_mode="HTML")
                return
        else:
            await message.reply("<b>❌ Данная команда доступна только для администрации</b>", parse_mode="HTML")
            return
        return await func(message, *args, **kwargs)
    return wrapper

class MemberTrackerMiddleware(BaseMiddleware):
    async def __call__(self, handler, event: Message, data):
        if event.chat.type in ('group', 'supergroup'):
            if event.from_user:
                user = event.from_user
                chat_id = event.chat.id
                conn = sqlite3.connect('reputation.db', check_same_thread=False)
                cursor = conn.cursor()
                cursor.execute('''INSERT INTO chat_users (chat_id, user_id, first_name, tg_username) 
                                  VALUES (?, ?, ?, ?)
                                  ON CONFLICT(chat_id, user_id) DO UPDATE SET 
                                  first_name = ?, tg_username = ?''',
                               (chat_id, user.id, user.first_name, user.username, user.first_name, user.username))
                conn.commit()
                conn.close()
        return await handler(event, data)

dp.message.middleware(MemberTrackerMiddleware())

def init_db():
    conn = sqlite3.connect('reputation.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS chat_users
                      (chat_id INTEGER, user_id INTEGER, first_name TEXT, tg_username TEXT, rep INTEGER DEFAULT 0, 
                      PRIMARY KEY (chat_id, user_id))''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS chat_settings
                      (chat_id INTEGER PRIMARY KEY, automute_enabled INTEGER DEFAULT 0, spam_status INTEGER DEFAULT 0,
                       spam_limit INTEGER DEFAULT 5, spam_mute_type TEXT DEFAULT 'full', has_spam_config INTEGER DEFAULT 0,
                       auto_punish INTEGER DEFAULT 0, punish_amount INTEGER DEFAULT 5, reports_enabled INTEGER DEFAULT 1,
                       spam_mute_time TEXT DEFAULT '01:00', regular_mute_type TEXT DEFAULT 'full', regular_mute_time TEXT DEFAULT '01:00',
                       low_rep_status INTEGER DEFAULT 0, low_rep_threshold INTEGER DEFAULT -10, low_rep_mute_time TEXT DEFAULT '01:00',
                       max_rep INTEGER DEFAULT 100, min_rep INTEGER DEFAULT -100, low_rep_mute_type TEXT DEFAULT 'full')''')
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN low_rep_status INTEGER DEFAULT 0')
    except sqlite3.OperationalError: pass
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN low_rep_threshold INTEGER DEFAULT -10')
    except sqlite3.OperationalError: pass
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN low_rep_mute_time TEXT DEFAULT "01:00"')
    except sqlite3.OperationalError: pass
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN max_rep INTEGER DEFAULT 100')
    except sqlite3.OperationalError: pass
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN min_rep INTEGER DEFAULT -100')
    except sqlite3.OperationalError: pass
    try: cursor.execute('ALTER TABLE chat_settings ADD COLUMN low_rep_mute_type TEXT DEFAULT "full"')
    except sqlite3.OperationalError: pass
    conn.commit()
    conn.close()

def get_settings(chat_id):
    conn = sqlite3.connect('reputation.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM chat_settings WHERE chat_id = ?', (chat_id,))
    res = cursor.fetchone()
    conn.close()
    if not res:
        conn = sqlite3.connect('reputation.db', check_same_thread=False)
        cursor = conn.cursor()
        cursor.execute('INSERT INTO chat_settings (chat_id) VALUES (?)', (chat_id,))
        conn.commit()
        conn.close()
        return (chat_id, 0, 0, 5, 'full', 0, 0, 5, 1, '01:00', 'full', '01:00', 0, -10, '01:00', 100, -100, 'full') 
    return res

def change_rep(chat_id, user_id, first_name, tg_username, amount):
    conn = sqlite3.connect('reputation.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute('SELECT rep FROM chat_users WHERE chat_id = ? AND user_id = ?', (chat_id, user_id))
    result = cursor.fetchone()
    current_rep = result[0] if result else 0
    
    settings = get_settings(chat_id)
    max_r, min_r = settings[15], settings[16]
    new_rep = max(min_r, min(max_r, current_rep + amount))
    
    cursor.execute('''INSERT INTO chat_users (chat_id, user_id, first_name, tg_username, rep) VALUES (?, ?, ?, ?, ?)
                      ON CONFLICT(chat_id, user_id) DO UPDATE SET first_name = ?, tg_username = ?, rep = ?''', 
                   (chat_id, user_id, first_name, tg_username, new_rep, first_name, tg_username, new_rep))
    conn.commit()
    conn.close()
    return new_rep, current_rep

init_db()

def get_status_table(chat_id):
    s = get_settings(chat_id)
    icon = "🟢 ВКЛЮЧЕННО" if s[2] == 1 else "🔴 ВЫКЛЮЧЕНО"
    spam_m = "полное лишение" if s[4] == 'full' else "лишение медиа"
    reg_m = "полное лишение" if s[10] == 'full' else "лишение медиа"
    return (f"<b>🛡️ Защита от спама: {icon}\n📊 Спам = {s[3]}\n⏱️ Мут за Спам: {s[9]}\n🔇 Тип (спам): {spam_m}\n\n"
            f"⏱️ Мут обычный: {s[11]}\n🔇 Тип (обычный): {reg_m}</b>")

def get_spam_status_table(chat_id):
    s = get_settings(chat_id)
    icon = "🟢 ВКЛЮЧЕННО" if s[2] == 1 else "🔴 ВЫКЛЮЧЕНО"
    spam_m = "полное лишение" if s[4] == 'full' else "лишение медиа"
    return f"<b>🛡️ Защита от спама: {icon}\n📊 Спам = {s[3]}\n⏱️ Мут за Спам: {s[9]}\n🔇 Тип (спам): {spam_m}</b>"

@dp.message(CommandStart())
async def cmd_start(message: Message):
    await message.answer("<b>👋 Привет! Я бот для изменения репутации и модерации.\nНапишите <u>/repcommands</u> для списка команд.</b>", parse_mode="HTML")

@dp.message(Command("repcommands"))
async def cmd_repcommands(message: Message):
    text = (
        "<b><u>📋 Доступные команды:</u>\n"
        "<u>/table</u> — Топ участников\n"
        "<u>/mutestatus</u> — Настройки авто-мута\n"
        "<u>/repreport</u> — Жалоба\n"
        "<u>/repmute</u> — Замутить (ответом)\n"
        "<u>/antirepmute</u> — Размутить (ответом)\n"
        "<u>/infofcommand</u> — Инфо о всех командах\n\n"
        "<u>⚙️ Настройки для админов:</u>\n"
        "<u>/automute</u> — Модуль авто-мута\n"
        "<u>/autodownrep</u> — Понижение репутации\n"
        "<u>/regrep</u> — Лимиты репутации\n"
        "<u>/onreport</u> / <u>/offreport</u> — Вкл/Выкл репорты</b>"
    )
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("infofcommand"))
async def cmd_infofcommand(message: Message):
    text = (
        "<b><u>📖 Информация о командах:</u>\n\n"
        "<u>/start</u> — Запускает бота.\n"
        "<u>/table</u> — Таблица репутации.\n"
        "<u>/repreport</u> — Жалоба владельцу.\n"
        "<u>/repmute</u> — Замутить нарушителя.\n"
        "<u>/antirepmute</u> — Размутить нарушителя.\n"
        "<u>/+[число]rep</u> — Повысить репутацию.\n"
        "<u>/-[число]rep</u> — Понизить репутацию.\n\n"
        "<u>🛠 Настройки:</u>\n"
        "<u>/automute</u> — Модуль авто-мута.\n"
        "<u>/autodownrep</u> — Понижение репутации.\n"
        "<u>/regrep [мин] [макс]</u> — Лимиты репутации.\n"
        "<u>/onreport</u> — Включить жалобы.\n"
        "<u>/offreport</u> — Выключить жалобы.\n\n"
        "<u>⚙️ Под-команды:</u>\n"
        "<code>/Мут защита от спама</code> | <code>/Мут обычный мут</code> | <code>/Мут за низкую репутацию</code>\n"
        "<code>/включить защита от спама</code> / <code>/выключить защита от спама</code>\n"
        "<code>/спам [число]</code>\n"
        "<code>/полное лишение защита от спама</code> / <code>/лишение стикеров и гиф защита от спама</code>\n"
        "<code>/полное лишение обычный мут</code> / <code>/лишение стикеров и гиф обычный мут</code>\n"
        "<code>/включить авто понижение репутации</code> / <code>/выключить авто понижение репутации</code>\n"
        "<code>/снижение [число]</code>\n"
        "<code>/время защита от спама [ЧЧ:ММ]</code>\n"
        "<code>/время обычный мут [ЧЧ:ММ]</code>\n"
        "<code>/Репмут -[число]</code>\n"
        "<code>/Репмут время [ЧЧ:ММ]</code>\n"
        "<code>/включить Мут за низкую репутацию</code> / <code>/выключить Мут за низкую репутацию</code>\n"
        "<code>/полное лишение мут за низкую репутацию</code> / <code>/лишение стикеров и гиф мут за низкую репутацию</code></b>"
    )
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("repreport"), F.reply_to_message)
async def cmd_repreport(message: Message):
    settings = get_settings(message.chat.id)
    if settings[8] == 0:
        await message.reply("<b>🔴 Репорты отключены</b>", parse_mode="HTML")
        return
    try:
        admins = await message.bot.get_chat_administrators(message.chat.id)
        creator = next((admin for admin in admins if admin.status == 'creator'), None)
        if not creator: return
        c_link = f"<a href='tg://user?id={creator.user.id}'>{html.escape(creator.user.first_name)}</a>"
        v_name = html.escape(message.reply_to_message.from_user.first_name)
        await message.reply(f"<b>🔔 {c_link}, поступила жалоба на {v_name} в нарушении правил.</b>", parse_mode="HTML")
    except Exception as e: print(f"Ошибка: {e}")

@dp.message(Command("onreport"))
async def cmd_onreport(message: Message):
    conn = sqlite3.connect('reputation.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute('UPDATE chat_settings SET reports_enabled = 1 WHERE chat_id = ?', (message.chat.id,))
    conn.commit(); conn.close()
    await message.answer("<b>🟢 Репорты включены</b>", parse_mode="HTML")

@dp.message(Command("offreport"))
async def cmd_offreport(message: Message):
    conn = sqlite3.connect('reputation.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute('UPDATE chat_settings SET reports_enabled = 0 WHERE chat_id = ?', (message.chat.id,))
    conn.commit(); conn.close()
    await message.answer("<b>🔴 Репорты выключены</b>", parse_mode="HTML")

@dp.message(F.sticker | F.animation, F.chat.type.in_({'group', 'supergroup'}))
async def anti_spam_stickers(message: Message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    try:
        member = await message.bot.get_chat_member(chat_id, user_id)
        if member.status in ['administrator', 'creator']: return
    except: pass
    settings = get_settings(chat_id)
    if settings[2] == 1: 
        now = time.time()
        if chat_id not in sticker_tracker: sticker_tracker[chat_id] = {}
        if user_id not in sticker_tracker[chat_id]:
            sticker_tracker[chat_id][user_id] = {'count': 1, 'messages': [message.message_id], 'time': now}
        else:
            track = sticker_tracker[chat_id][user_id]
            if now - track['time'] < 15:
                track['count'] += 1
                track['messages'].append(message.message_id)
                track['time'] = now
            else:
                sticker_tracker[chat_id][user_id] = {'count': 1, 'messages': [message.message_id], 'time': now}
        if sticker_tracker[chat_id][user_id]['count'] >= settings[3]:
            for msg_id in sticker_tracker[chat_id][user_id]['messages']:
                try: await message.bot.delete_message(chat_id, msg_id)
                except: pass
            try:
                if settings[4] == 'full':
                    perms = ChatPermissions(can_send_messages=False, can_send_audios=False, can_send_documents=False, can_send_photos=False, can_send_videos=False, can_send_video_notes=False, can_send_voice_notes=False, can_send_polls=False, can_send_other_messages=False, can_add_web_page_previews=False)
                else:
                    perms = ChatPermissions(can_send_messages=True, can_send_audios=False, can_send_documents=False, can_send_photos=False, can_send_videos=False, can_send_video_notes=False, can_send_voice_notes=False, can_send_polls=False, can_send_other_messages=False, can_add_web_page_previews=False)
                
                time_str = settings[9]
                h, m = map(int, time_str.split(':'))
                until_date = int(time.time() + (h * 3600) + (m * 60))
                await message.bot.restrict_chat_member(chat_id, user_id, permissions=perms, until_date=until_date)
                user_link = f"<a href='tg://user?id={user_id}'>{html.escape(message.from_user.first_name)}</a>"
                await message.bot.send_message(chat_id, f"<b>🚫 Я замутил {user_link} по причине спам на {time_str}</b>", parse_mode="HTML")
                if settings[6] == 1: change_rep(chat_id, user_id, message.from_user.first_name, message.from_user.username, -settings[7])
            except Exception as e: print(f"Ошибка мута: {e}")
            sticker_tracker[chat_id][user_id] = {'count': 0, 'messages': [], 'time': now}

@dp.message(F.chat.type.in_({'group', 'supergroup'}), F.text, ~F.text.startswith('/'))
async def check_low_rep_on_message(message: Message):
    try:
        member = await message.bot.get_chat_member(message.chat.id, message.from_user.id)
        if member.status in ['administrator', 'creator']: return
    except: pass
    settings = get_settings(message.chat.id)
    if settings[12] == 1: 
        conn = sqlite3.connect('reputation.db', check_same_thread=False)
        cursor = conn.cursor()
        cursor.execute('SELECT rep FROM chat_users WHERE chat_id = ? AND user_id = ?', (message.chat.id, message.from_user.id))
        res = cursor.fetchone(); conn.close()
        current_rep = res[0] if res else 0
        if current_rep <= settings[13]:
            try:
                time_str = settings[14]
                mute_type = settings[17]
                h, m = map(int, time_str.split(':'))
                until_date = int(time.time() + (h * 3600) + (m * 60))
                
                if mute_type == 'full':
                    perms = ChatPermissions(can_send_messages=False, can_send_audios=False, can_send_documents=False, can_send_photos=False, can_send_videos=False, can_send_video_notes=False, can_send_voice_notes=False, can_send_polls=False, can_send_other_messages=False, can_add_web_page_previews=False)
                else:
                    perms = ChatPermissions(can_send_messages=True, can_send_audios=False, can_send_documents=False, can_send_photos=False, can_send_videos=False, can_send_video_notes=False, can_send_voice_notes=False, can_send_polls=False, can_send_other_messages=False, can_add_web_page_previews=False)
                    
                await message.bot.restrict_chat_member(message.chat.id, message.from_user.id, permissions=perms, until_date=until_date)
                u_link = f"<a href='tg://user?id={message.from_user.id}'>{html.escape(message.from_user.first_name)}</a>"
                await message.answer(f"<b>🚫 {u_link} замучен на {time_str} за низкую репутацию ({current_rep}).</b>", parse_mode="HTML")
            except: pass

@dp.message(F.chat.type.in_({'group', 'supergroup'}), ~F.text, ~F.sticker, ~F.animation)
async def block_other_media(message: Message):
    if message.text: return
    try:
        member = await message.bot.get_chat_member(message.chat.id, message.from_user.id)
        if member.status in ['administrator', 'creator']: return
    except: pass
    try:
        await message.delete()
        perms = ChatPermissions(can_send_messages=True, can_send_audios=False, can_send_documents=False, can_send_photos=False, can_send_videos=False, can_send_video_notes=False, can_send_voice_notes=False, can_send_polls=False, can_send_other_messages=False, can_add_web_page_previews=False)
        await message.bot.restrict_chat_member(message.chat.id, message.from_user.id, permissions=perms)
    except Exception as e: print(f"Ошибка медиа: {e}")

@dp.message(F.text.startswith("/automute") | F.text.startswith("/automute@"))
@admin_only
async def cmd_automute(message: Message):
    conn = sqlite3.connect('reputation.db', check_same_thread=False); cursor = conn.cursor()
    cursor.execute('INSERT OR IGNORE INTO chat_settings (chat_id) VALUES (?)', (message.chat.id,))
    cursor.execute('UPDATE chat_settings SET automute_enabled = 1 WHERE chat_id = ?', (message.chat.id,))
    conn.commit(); conn.close()
    await message.answer("<b><u>⚙️ Выберите категорию настроек:</u>\n- Защита от спама\n- Обычный мут\n- Мут за низкую репутацию\n\n(Выбрать: <code>/Мут [название]</code>)</b>", parse_mode="HTML")

@dp.message(F.text.lower() == "/мут обычный мут")
@admin_only
async def cmd_mut_regular(message: Message):
    await message.answer("<b><u>⚙️ Настройки обычного мута:</u>\n\n"
                         "🔇 Тип: <code>/[полное лишение/лишение стикеров и гиф] обычный мут</code>\n"
                         "⏱️ Время: <code>/время обычный мут [ЧЧ:ММ]</code>\n\n"
                         "🚫 Замутить: <code>/repmute</code> (ответом на сообщение)</b>", parse_mode="HTML")

@dp.message(F.text.lower().startswith("/мут за низкую репутацию"))
@admin_only
async def cmd_mut_low_rep(message: Message):
    s = get_settings(message.chat.id)
    icon = "🟢 ВКЛЮЧЕННО" if s[12] == 1 else "🔴 ВЫКЛЮЧЕНО"
    low_m = "полное лишение" if s[17] == 'full' else "лишение медиа"
    text = (f"<b><u>⚙️ Настройки мута за низкую репутацию:</u>\n\n"
            f"📉 Статус: {icon}\n"
            f"🔄 Переключить: <code>/[включить/выключить] мут за низкую репутацию</code>\n\n"
            f"📊 Текущий порог: {s[13]}\n"
            f"⚙️ Изменить порог: <code>/Репмут -[число]</code>\n\n"
            f"⏱️ Текущее время: {s[14]}\n"
            f"⏱️ Изменить время: <code>/Репмут время [ЧЧ:ММ]</code>\n\n"
            f"🔇 Текущий тип: {low_m}\n"
            f"⚙️ Изменить тип: <code>/[полное лишение/лишение стикеров и гиф] мут за низкую репутацию</code></b>")
    await message.answer(text, parse_mode="HTML")

@dp.message(F.text.lower() == "/включить мут за низкую репутацию")
@admin_only
async def cmd_enable_low_rep(message: Message):
    conn = sqlite3.connect('reputation.db', check_same_thread=False); cursor = conn.cursor()
    cursor.execute('UPDATE chat_settings SET low_rep_status = 1 WHERE chat_id = ?', (message.chat.id,)); conn.commit(); conn.close()
    await message.answer("<b>🟢 Мут за низкую репутацию включен.</b>", parse_mode="HTML")

@dp.message(F.text.lower() == "/выключить мут за низкую репутацию")
@admin_only
async def cmd_disable_low_rep(message: Message):
    conn = sqlite3.connect('reputation.db', check_same_thread=False); cursor = conn.cursor()
    cursor.execute('UPDATE chat_settings SET low_rep_status = 0 WHERE chat_id = ?', (message.chat.id,)); conn.commit(); conn.close()
    await message.answer("<b>🔴 Мут за низкую репутацию выключен.</b>", parse_mode="HTML")

@dp.message(F.text.lower().startswith("/репмут -
