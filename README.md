from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackQueryHandler, ConversationHandler
import logging
import sqlite3

# Логирование
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# База данных для хранения заказов
conn = sqlite3.connect('orders.db', check_same_thread=False)
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS orders
                  (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, service_type TEXT, details TEXT, status TEXT)''')
conn.commit()

# Состояния для ConversationHandler
CHOOSING, TYPING_DETAILS, CONFIRMATION, PAYMENT, UPLOAD_RECEIPT = range(5)

# Токен бота
TOKEN = '8179699819:AAF_0MSWN2P1lLGIah71Hawwofk9luCtXBo'

# Главное меню
def start(update: Update, context) -> int:
    keyboard = [
        [InlineKeyboardButton("Хизматлар", callback_data='services')],
        [InlineKeyboardButton("Буюртмалар тарихи", callback_data='order_history')],
        [InlineKeyboardButton("Ҳамкорлик", callback_data='collaboration')],
        [InlineKeyboardButton("Мурожаатлар", callback_data='contact')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Асосий меню:', reply_markup=reply_markup)
    return CHOOSING

# Обработка выбора услуги
def services(update: Update, context) -> int:
    query = update.callback_query
    query.answer()
    keyboard = [
        [InlineKeyboardButton("Реферат тайёрлаш", callback_data='referat')],
        [InlineKeyboardButton("Матн териш", callback_data='text_typing')],
        [InlineKeyboardButton("Тақдимотлар тайёрлаш (слайд)", callback_data='presentation')],
        [InlineKeyboardButton("Онлайн авто суғурта", callback_data='auto_insurance')],
        [InlineKeyboardButton("Солиқ ҳисоботларини юбориш", callback_data='tax_reports')],
        [InlineKeyboardButton("Солиқ декларацияларини юбориш", callback_data='tax_declaration')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    query.edit_message_text(text="Хизматлар:", reply_markup=reply_markup)
    return CHOOSING

# Обработка заказа реферата
def referat(update: Update, context) -> int:
    query = update.callback_query
    query.answer()
    keyboard = [
        [InlineKeyboardButton("15 вароқ", callback_data='referat_15')],
        [InlineKeyboardButton("15 вароқдан кўп", callback_data='referat_more')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    query.edit_message_text(text="Реферат ҳажмини танланг:", reply_markup=reply_markup)
    return CHOOSING

# Обработка выбора объема реферата
def referat_volume(update: Update, context) -> int:
    query = update.callback_query
    query.answer()
    context.user_data['service_type'] = 'Реферат'
    if query.data == 'referat_15':
        context.user_data['price'] = 50000
        context.user_data['volume'] = '15 вароқ'
    else:
        context.user_data['price'] = 70000
        context.user_data['volume'] = '15 вароқдан кўп'
    query.edit_message_text(text=f"Реферат ҳажми: {context.user_data['volume']}\nНархи: {context.user_data['price']} сўм\nТасдиқлаш учун 'Тасдиқлаш' тугмасини босинг.")
    return CONFIRMATION

# Обработка подтверждения заказа
def confirm_order(update: Update, context) -> int:
    query = update.callback_query
    query.answer()
    query.edit_message_text(text="Реферат тайёрлаш учун маълумотларни киритинг:\n1. Реферат мавзуси\n2. Йўналиш ва босқич\n3. Фан ва гуруҳ\n4. Устознинг Ф.И.Ш.")
    return TYPING_DETAILS

# Обработка ввода деталей заказа
def save_details(update: Update, context) -> int:
    user_data = context.user_data
    user_data['details'] = update.message.text
    update.message.reply_text("Маълумотлар сақланди. Тўлов қилиш учун 'Тўлов қилиш' тугмасини босинг.")
    return PAYMENT

# Обработка оплаты
def payment(update: Update, context) -> int:
    query = update.callback_query
    query.answer()
    query.edit_message_text(text="Тўлов қилиш учун карта рақами: 8600*********6648\nТўлов қилгандан сўнг 'Тўлов қилдим' тугмасини босинг.")
    return UPLOAD_RECEIPT

# Обработка загрузки чека
def upload_receipt(update: Update, context) -> int:
    user_data = context.user_data
    user_data['receipt'] = update.message.photo[-1].file_id
    update.message.reply_text("Чек юкланди. Буюртма рақамингиз: 12345\nБуюртма ҳолатини 'Буюртмалар тарихи'дан кўриб туришингиз мумкин.")
    cursor.execute("INSERT INTO orders (user_id, service_type, details, status) VALUES (?, ?, ?, ?)",
                   (update.message.from_user.id, user_data['service_type'], user_data['details'], 'Тўлов кутилмокда'))
    conn.commit()
    return ConversationHandler.END

# Обработка истории заказов
def order_history(update: Update, context) -> None:
    cursor.execute("SELECT * FROM orders WHERE user_id = ?", (update.message.from_user.id,))
    orders = cursor.fetchall()
    if orders:
        for order in orders:
            update.message.reply_text(f"Буюртма рақами: {order[0]}\nХизмат: {order[2]}\nҲолат: {order[4]}")
    else:
        update.message.reply_text("Сизда буюртмалар мавжуд эмас.")

# Обработка отмены
def cancel(update: Update, context) -> int:
    update.message.reply_text('Буюртма бекор қилинди.')
    return ConversationHandler.END

def main() -> None:
    updater = Updater(TOKEN)
    dispatcher = updater.dispatcher

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            CHOOSING: [
                CallbackQueryHandler(services, pattern='^services$'),
                CallbackQueryHandler(referat, pattern='^referat$'),
                CallbackQueryHandler(referat_volume, pattern='^referat_15$|^referat_more$'),
                CallbackQueryHandler(confirm_order, pattern='^confirm$')
            ],
            TYPING_DETAILS: [MessageHandler(Filters.text & ~Filters.command, save_details)],
            CONFIRMATION: [CallbackQueryHandler(confirm_order, pattern='^confirm$')],
            PAYMENT: [CallbackQueryHandler(payment, pattern='^payment$')],
            UPLOAD_RECEIPT: [MessageHandler(Filters.photo, upload_receipt)]
        },
        fallbacks=[CommandHandler('cancel', cancel)]
    )

    dispatcher.add_handler(conv_handler)
    dispatcher.add_handler(CommandHandler('order_history', order_history))

    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()
