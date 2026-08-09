# tg-shop-bot
import telebot
from telebot import types

# ==========================================
# 1. ОСНОВНЫЕ НАСТРОЙКИ
# ==========================================
TOKEN = '8923375368:AAEaQ8UJwVlpxTEU1TUUJv4j3thYBEZ1RRA'
ADMIN_USERNAME = '@nifraga1'  # На этот ник будут идти заказы

bot = telebot.TeleBot(TOKEN)

# ==========================================
# 2. БАЗА ДАННЫХ КАТАЛОГА (Категории и Товары)
# ==========================================
CATALOG = {
    "sneakers": {
        "title": "👟 Кроссовки",
        "items": {
            "nike_jordan": {
                "name": "Nike Air Jordan 1",
                "price": "5 900 руб.",
                "desc": "Легендарные кроссовки. Натуральная кожа, премиальное качество.",
                "photo": "https://images.unsplash.com/photo-1552346154-21d32810aba3?w=500"
            },
            "adidas_forum": {
                "name": "Adidas Forum Low",
                "price": "4 800 руб.",
                "desc": "Удобная классическая модель для ежедневной носки.",
                "photo": "https://images.unsplash.com/photo-1542291026-7eec264c27ff?w=500"
            }
        }
    },
    "clothes": {
        "title": "👕 Одежда",
        "items": {
            "hoodie_travis": {
                "name": "Худи Travis Scott Oversize",
                "price": "3 800 руб.",
                "desc": "Плотный 100% хлопок, свободный крой и яркий принт.",
                "photo": "https://images.unsplash.com/photo-1509967419530-da38b4704bc6?w=500"
            }
        }
    }
}

# ==========================================
# 3. ГЛАВНОЕ МЕНЮ (/start)
# ==========================================
@bot.message_handler(commands=['start'])
def start_command(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    btn_catalog = types.KeyboardButton("🛍 Каталог товаров")
    btn_info = types.KeyboardButton("ℹ️ О магазине")
    btn_support = types.KeyboardButton("📞 Связь с менеджером")
    
    markup.add(btn_catalog)
    markup.add(btn_info, btn_support)

    user_name = message.from_user.first_name
    welcome_text = (
        f"Привет, {user_name}! 👋\n\n"
        f"Добро пожаловать в **StreetWear Store** — магазин премиум одежды и обуви!\n\n"
        f"Воспользуйся меню ниже, чтобы открыть каталог и выбрать товар 👇"
    )
    bot.send_message(message.chat.id, welcome_text, reply_markup=markup, parse_mode="Markdown")

# ==========================================
# 4. ОБРАБОТКА КНОПОК ГЛАВНОГО МЕНЮ
# ==========================================
@bot.message_handler(content_types=['text'])
def handle_menu(message):
    # Выбор категории
    if message.text == "🛍 Каталог товаров":
        inline_markup = types.InlineKeyboardMarkup()
        
        # Генерируем кнопки категорий автоматически
        for cat_id, cat_data in CATALOG.items():
            btn = types.InlineKeyboardButton(
                text=cat_data["title"], 
                callback_data=f"cat_{cat_id}"
            )
            inline_markup.add(btn)
            
        bot.send_message(
            message.chat.id, 
            "📂 **Выбери категорию товаров:**", 
            reply_markup=inline_markup, 
            parse_mode="Markdown"
        )

    # Информация
    elif message.text == "ℹ️ О магазине":
        info_text = (
            "🏪 **О магазине StreetWear Store:**\n\n"
            "• Быстрая доставка по СНГ (3-5 дней).\n"
            "• Гарантия возврата и примерка перед покупкой.\n"
            "• Более 1000+ живых отзывов клиентов."
        )
        bot.send_message(message.chat.id, info_text, parse_mode="Markdown")

    # Связь
    elif message.text == "📞 Связь с менеджером":
        support_text = f"📩 По всем вопросам и для заказа пишите менеджеру: {ADMIN_USERNAME}"
        bot.send_message(message.chat.id, support_text)

# ==========================================
# 5. ИНЛАЙН-НАВИГАЦИЯ (Категории, Товары, Заказ)
# ==========================================
@bot.callback_query_handler(func=lambda call: True)
def handle_inline(call):
    # 5.1. Пользователь выбрал категорию
    if call.data.startswith("cat_"):
        cat_id = call.data.replace("cat_", "")
        category = CATALOG[cat_id]
        
        bot.send_message(call.message.chat.id, f"📦 **Категория: {category['title']}**", parse_mode="Markdown")
        
        # Отправляем каждый товар из этой категории отдельной карточкой
        for item_id, item_data in category["items"].items():
            inline_markup = types.InlineKeyboardMarkup()
            buy_btn = types.InlineKeyboardButton(
                text=f"🛒 Купить ({item_data['price']})", 
                callback_data=f"buy_{cat_id}_{item_id}"
            )
            inline_markup.add(buy_btn)
            
            caption = f"**{item_data['name']}**\n\n{item_data['desc']}\n\n💰 **Цена:** {item_data['price']}"
            
            bot.send_photo(
                call.message.chat.id, 
                photo=item_data["photo"], 
                caption=caption, 
                parse_mode="Markdown", 
                reply_markup=inline_markup
            )

    # 5.2. Пользователь нажал "Купить" под конкретным товаром
    elif call.data.startswith("buy_"):
        bot.answer_callback_query(call.id, text="Заявка сформирована!", show_alert=True)
        
        # Разбираем ID категории и товара
        parts = call.data.split("_")
        cat_id = parts[1]
        item_id = parts[2]
        item_data = CATALOG[cat_id]["items"][item_id]
        
        order_success_msg = (
            f"✅ **Заказ принят!**\n\n"
            f"Товар: **{item_data['name']}**\n"
            f"К оплате: **{item_data['price']}**\n\n"
            f"Для подтверждения и оформления доставки напишите нашему менеджеру: {ADMIN_USERNAME}"
        )
        bot.send_message(call.message.chat.id, order_success_msg, parse_mode="Markdown")

# ==========================================
# 6. ЗАПУСК СЕРВЕРА
# ==========================================
print("🚀 Бот-Каталог успешно запущен!")
bot.polling(none_stop=True)
