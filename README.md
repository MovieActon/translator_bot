import telebot
import psycopg2
# import your_payment_gateway_library  # If needed

# --- Configuration ---
BOT_TOKEN = "[YOUR_BOT_TOKEN]"
ADMIN_ID = "[YOUR_ADMIN_ID]"
CASHFREE_API_KEY = "[YOUR_CASHFREE_API_KEY]"
CASHFREE_SECRET_KEY = "[YOUR_CASHFREE_SECRET_KEY]"
CASHFREE_WEBHOOK_URL = "[YOUR_CASHFREE_WEBHOOK_URL]"
DATABASE_URL = "your_postgresql_connection_string" # Replace with your actual connection string

bot = telebot.TeleBot(BOT_TOKEN)

# --- Database Functions ---
def connect_db():
    """Connect to the PostgreSQL database."""
    conn = psycopg2.connect(DATABASE_URL)
    return conn

def create_tables():
    """Create necessary tables in the database if they don't exist."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS movies (
            id SERIAL PRIMARY KEY,
            title VARCHAR(255) NOT NULL,
            thumbnail VARCHAR(255),
            hd_link VARCHAR(255),
            hd_price INTEGER,
            medium_link VARCHAR(255),
            medium_price INTEGER,
            low_link VARCHAR(255),
            low_price INTEGER,
            is_free BOOLEAN DEFAULT FALSE,
            category VARCHAR(50),
            added_at TIMESTAMP DEFAULT NOW()
        );
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS web_series (
            id SERIAL PRIMARY KEY,
            title VARCHAR(255) NOT NULL,
            thumbnail VARCHAR(255),
            hd_link VARCHAR(255),
            hd_price INTEGER,
            is_free BOOLEAN DEFAULT FALSE,
            category VARCHAR(50),
            added_at TIMESTAMP DEFAULT NOW()
        );
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id BIGINT PRIMARY KEY,
            # Add other user-related information if needed
        );
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS referrals (
            id SERIAL PRIMARY KEY,
            referrer_id BIGINT NOT NULL,
            referred_user_id BIGINT NOT NULL,
            movie_id INTEGER,
            web_series_id INTEGER,
            referred_at TIMESTAMP DEFAULT NOW(),
            FOREIGN KEY (referrer_id) REFERENCES users(id),
            FOREIGN KEY (referred_user_id) REFERENCES users(id),
            FOREIGN KEY (movie_id) REFERENCES movies(id),
            FOREIGN KEY (web_series_id) REFERENCES web_series(id),
            UNIQUE (referrer_id, referred_user_id, movie_id, web_series_id)
        );
    """)
    conn.commit()
    cursor.close()
    conn.close()

def add_movie_to_db(title, thumbnail, hd_link, hd_price, medium_link, medium_price, low_link, low_price, is_free, category):
    """Add a movie to the database."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO movies (title, thumbnail, hd_link, hd_price, medium_link, medium_price, low_link, low_price, is_free, category)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
    """, (title, thumbnail, hd_link, hd_price, medium_link, medium_price, low_link, low_price, is_free, category))
    conn.commit()
    cursor.close()
    conn.close()

def add_web_series_to_db(title, thumbnail, hd_link, hd_price, is_free, category):
    """Add a web series to the database."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO web_series (title, thumbnail, hd_link, hd_price, is_free, category)
        VALUES (%s, %s, %s, %s, %s, %s);
    """, (title, thumbnail, hd_link, hd_price, is_free, category))
    conn.commit()
    cursor.close()
    conn.close()

def get_movies_by_category(category, is_free=False, order_by='added_at DESC'):
    """Fetch movies by category from the database."""
    conn = connect_db()
    cursor = conn.cursor()
    query = f"SELECT * FROM movies WHERE category = %s AND is_free = %s ORDER BY {order_by};"
    cursor.execute(query, (category, is_free))
    movies = cursor.fetchall()
    cursor.close()
    conn.close()
    return movies

def get_web_series_by_category(category, is_free=False, order_by='added_at DESC'):
    """Fetch web series by category from the database."""
    conn = connect_db()
    cursor = conn.cursor()
    query = f"SELECT * FROM web_series WHERE category = %s AND is_free = %s ORDER BY {order_by};"
    cursor.execute(query, (category, is_free))
    web_series = cursor.fetchall()
    cursor.close()
    conn.close()
    return web_series

def search_movie_by_name(name):
    """Search for a movie by name."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM movies WHERE title ILIKE %s;", (f"%{name}%",))
    movies = cursor.fetchall()
    cursor.close()
    conn.close()
    return movies

def search_web_series_by_name(name):
    """Search for a web series by name."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM web_series WHERE title ILIKE %s;", (f"%{name}%",))
    web_series = cursor.fetchall()
    cursor.close()
    conn.close()
    return web_series

def get_popular_movies():
    """Fetch all popular movies (you'll need a way to define 'popular')."""
    # This is a placeholder - you might need to add a 'popularity' field or logic
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM movies ORDER BY added_at DESC;") # Example: Order by recently added
    movies = cursor.fetchall()
    cursor.close()
    conn.close()
    return movies

def get_popular_web_series():
    """Fetch all popular web series (you'll need a way to define 'popular')."""
    # This is a placeholder - you might need to add a 'popularity' field or logic
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM web_series ORDER BY added_at DESC;") # Example: Order by recently added
    web_series = cursor.fetchall()
    cursor.close()
    conn.close()
    return web_series

def delete_movie(title):
    """Delete a movie from the database."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM movies WHERE title = %s;", (title,))
    conn.commit()
    cursor.close()
    conn.close()

def delete_web_series(title):
    """Delete a web series from the database."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM web_series WHERE title = %s;", (title,))
    conn.commit()
    cursor.close()
    conn.close()

# --- Bot Handlers ---

@bot.message_handler(commands=['start'])
def send_welcome(message):
    """Handles the /start command."""
    markup = telebot.types.ReplyKeyboardMarkup(row_width=2)
    item1 = telebot.types.KeyboardButton('🟢 New Latest Movie')
    item2 = telebot.types.KeyboardButton('🟢 Web Series')
    item3 = telebot.types.KeyboardButton('🟢 Popular Movie')
    item4 = telebot.types.KeyboardButton('🟢 Oldest Movie')
    item5 = telebot.types.KeyboardButton('🟢 Free Movie')
    markup.add(item5, item2, item1, item3, item4) # Free always on top
    welcome_message = """Welcome P-trailer Movie

⭐ Browse new latest movies or web series by searching or typing the button below:
[ 🟢 New Latest Movie ] or [ 🟢 Web Series ]

⭐ Browse popular movies or web series by searching or typing the button below:
[ 🟢 Popular Movie ] or [ 🟢 Web Series ]

⭐ Browse oldest movies or web series by searching or typing the button below:
[ 🟢 Oldest Movie ] or [ 🟢 Web Series ]

⭐ Browse free movies or web series by searching or typing the button below:
[ 🟢 Free Movie ] or [ 🟢 Web Series ]

⭐ Browse free movies or web series by searching or typing the button below:
👇
[ 🟢 New Latest Movie ] or [ 🟢 Web Series ]
[ 🟢 Popular Movie ] or [ 🟢 Web Series ]
[ 🟢 Oldest Movie ] or [ 🟢 Web Series ]
"""
    bot.send_message(message.chat.id, welcome_message, reply_markup=markup)

@bot.message_handler(func=lambda message: message.text == '🟢 New Latest Movie')
def handle_new_latest_movie(message):
    """Handles the 'New Latest Movie' button."""
    movies = get_movies_by_category('new_latest', is_free=True) # Show free first as per requirement
    if not movies:
        movies.extend(get_movies_by_category('new_latest', is_free=False))
    send_movie_list(message.chat.id, movies)

@bot.message_handler(func=lambda message: message.text == '🟢 Web Series')
def handle_web_series(message):
    """Handles the 'Web Series' button."""
    web_series = get_web_series_by_category('popular', is_free=True) # Assuming free popular web series
    if not web_series:
        web_series.extend(get_web_series_by_category('popular', is_free=False))
    send_web_series_list(message.chat.id, web_series) # Adjust category as needed

@bot.message_handler(func=lambda message: message.text == '🟢 Popular Movie')
def handle_popular_movie(message):
    """Handles the 'Popular Movie' button."""
    movies = get_movies_by_category('popular', is_free=True)
    if not movies:
        movies.extend(get_movies_by_category('popular', is_free=False))
    send_movie_list(message.chat.id, movies)

@bot.message_handler(func=lambda message: message.text == '🟢 Oldest Movie')
def handle_oldest_movie(message):
    """Handles the 'Oldest Movie' button."""
    movies = get_movies_by_category('oldest', is_free=True, order_by='added_at ASC')
    if not movies:
        movies.extend(get_movies_by_category('oldest', is_free=False, order_by='added_at ASC'))
    send_movie_list(message.chat.id, movies)

@bot.message_handler(func=lambda message: message.text == '🟢 Free Movie')
def handle_free_movie(message):
    """Handles the 'Free Movie' button."""
    movies = get_movies_by_category(None, is_free=True) # Fetch all free movies
    send_movie_list(message.chat.id, movies)

def send_movie_list(chat_id, movies):
    """Sends a list of movies to the user."""
    for movie in movies:
        markup = telebot.types.InlineKeyboardMarkup(row_width=1)
        share_button = telebot.types.InlineKeyboardButton("Share", switch_inline_query=movie[1]) # Example share text
        markup.add(share_button)
        thumbnail_url = movie[2] if movie[2] else "default_thumbnail_url" # Replace with default URL
        caption = f"*{movie[1]}*\n\n"
        if movie[8]: # is_free
            caption += f"Free HD Quality Movie Link: - {movie[3]}\n"
        else:
            caption += f"Pay {movie[4]} rupees HD Quality Movie Link: - {movie[3]}\n"
            caption += f"Pay {movie[6]} rupees Medium Quality Movie Link: - {movie[5]}\n"
            caption += f"Pay {movie[8]} rupees Low Quality Movie Link: - {movie[7]}\n\n"
            caption += "Refer this movie to five people and get it for free in any quality.\n"
        try:
            bot.send_photo(chat_id, thumbnail_url, caption=caption, parse_mode='Markdown', reply_markup=markup)
        except Exception as e:
            bot.send_message(chat_id, caption + f"\n(Thumbnail could not be loaded: {e})", parse_mode='Markdown', reply_markup=markup)

def send_web_series_list(chat_id, web_series):
    """Sends a list of web series to the user."""
    for series in web_series:
        markup = telebot.types.InlineKeyboardMarkup(row_width=1)
        share_button = telebot.types.InlineKeyboardButton("Share", switch_inline_query=series[1]) # Example share text
        markup.add(share_button)
        thumbnail_url = series[2] if series[2] else "default_thumbnail_url" # Replace with default URL
        caption = f"*{series[1]}*\n\n"
        if series[5]: # is_free
            caption += f"Free HD Quality Web Series Link: - {series[3]}\n"
        else:
            caption += f"Pay {series[4]} rupees HD Quality Web Series Link: - {series[3]}\n"
        try:
            bot.send_photo(chat_id, thumbnail_url, caption=caption, parse_mode='Markdown', reply_markup=markup)
        except Exception as e:
            bot.send_message(chat_id, caption + f"\n(Thumbnail could not be loaded: {e})", parse_mode='Markdown', reply_markup=markup)

@bot.message_handler(func=lambda message: message.text == 'add movie' and str(message.from_user.id) == ADMIN_ID)
def handle_add_movie_thumbnail(message):
    """Handles the 'add movie' command (admin)."""
    bot.send_message(message.chat.id, "Please send the movie thumbnail.")
    bot.register_next_step_handler(message, get_movie_name)

def get_movie_name(message):
    """Gets the movie name from the admin."""
    global movie_data
    movie_data = {'thumbnail': message.photo[0].file_id if message.photo else message.text} # Handle both photo and URL
    bot.send_message(message.chat.id, "Please enter the movie name.")
    bot.register_next_step_handler(message, get_hd_link_price)

def get_hd_link_price(message):
    """Gets the HD link and price from the admin."""
    global movie_data
    movie_data['title'] = message.text
    bot.send_message(message.chat.id, "Please enter the HD Quality movie link and set the price (e.g., link|price).")
    bot.register_next_step_handler(message, get_medium_link_price)

def get_medium_link_price(message):
    """Gets the Medium link and price from the admin."""
    global movie_data
    try:
        link, price = message.text.split('|')
        movie_data['hd_link'] = link.strip()
        movie_data['hd_price'] = int(price.strip())
    except ValueError:
        bot.send_message(message.chat.id, "Invalid format. Please use link|price.")
        bot.register_next_step_handler(message, get_medium_link_price)
        return
    bot.send_message(message.chat.id, "Please enter the Medium Quality movie link and set the price (e.g., link|price).")
    bot.register_next_step_handler(message, get_low_link_price)

def get_low_link_price(message):
    """Gets the Low link and price from the admin."""
    global movie_data
    try:
        link, price = message.text.split('|')
        movie_data['medium_link'] = link.strip()
        movie_data['medium_price'] = int(price.strip())
    except ValueError:
        bot.send_message(message.chat.id, "Invalid format. Please use link|price.")
        bot.register_next_step_handler(message, get_low_link_price)
        return
    bot.send_message(message.chat.id, "Please enter the Low Quality movie link and set the price (e.g., link|price).")
    bot.register_next_step_handler(message, get_category)

def get_category(message):
    """Gets the category of the movie."""
    global movie_data
    try:
        link, price = message.text.split('|')
        movie_data['low_link'] = link.strip()
        movie_data['low_price'] = int(price.strip())
    except ValueError:
        bot.send_message(message.chat.id, "Invalid format. Please use link|price.")
        bot.register_next_step_handler(message, get_category)
        return
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add('New Latest', 'Popular', 'Oldest')
    bot.send_message(message.chat.id, "Select the movie category:", reply_markup=markup)
    bot.register_next_step_handler(message, save_movie)

def save_movie(message):
    """Saves the movie data to the database."""
    global movie_data
    category = message.text.lower().replace(" ", "_")
    # Get file_id of the thumbnail
    file_info = bot.get_file(movie_data['thumbnail'])
    thumbnail_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
    add_movie_to_db(movie_data['title'], thumbnail_url, movie_data['hd_link'], movie_data['hd_price'],
                    movie_data['medium_link'], movie_data['medium_price'], movie_data['low_link'], movie_data['low_price'], False, category)
    bot.send_message(message.chat.id, f"Movie '{movie_data['title']}' added successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())
    send_new_movie_notification(movie_data['title']) # Send notification

def send_new_movie_notification(movie_title):
    """Sends a notification to all users about a new movie."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT id FROM users;") # You'll need to manage user IDs
    users = cursor.fetchall()
    cursor.close()
    conn.close()
    for user in users:
        try:
            bot.send_message(user[0], f"🎬 A new movie '{movie_title}' has been added! Check it out.")
        except Exception as e:
            print(f"Could not send notification to user {user[0]}: {e}")

@bot.message_handler(func=lambda message: message.text == 'add web series' and str(message.from_user.id) == ADMIN_ID)
def handle_add_web_series_thumbnail(message):
    """Handles the 'add web series' command (admin)."""
    bot.send_message(message.chat.id, "Please send the web series thumbnail.")
    bot.register_next_step_handler(message, get_web_series_name)

def get_web_series_name(message):
    """Gets the web series name from the admin."""
    global web_series_data
    web_series_data = {'thumbnail': message.photo[0].file_id if message.photo else message.text} # Handle both photo and URL
    bot.send_message(message.chat.id, "Please enter the web series name.")
    bot.register_next_step_handler(message, get_web_series_hd_link_price)

def get_web_series_hd_link_price(message):
    """Gets the HD link and price for the web series from the admin."""
    global web_series_data
    web_series_data['title'] = message.text
    bot.send_message(message.chat.id, "Please enter the HD Quality web series link and set the price (e.g., link|price).")
    bot.register_next_step_handler(message, get_web_series_category)

def get_web_series_category(message):
    """Gets the category of the web series."""
    global web_series_data
    try:
        link, price = message.text.split('|')
        web_series_data['hd_link'] = link.strip()
        web_series_data['hd_price'] = int(price.strip())
    except ValueError:
        bot.send_message(message.chat.id, "Invalid format. Please use link|price.")
        bot.register_next_step_handler(message, get_web_series_category)
        return
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add('New Latest', 'Popular', 'Oldest')
    bot.send_message(message.chat.id, "Select the web series category:", reply_markup=markup)
    bot.register_next_step_handler(message, save_web_series)

def save_web_series(message):
    """Saves the web series data to the database."""
    global web_series_data
    category = message.text.lower().replace(" ", "_")
    # Get file_id of the thumbnail
    file_info = bot.get_file(web_series_data['thumbnail'])
    thumbnail_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
    add_web_series_to_db(web_series_data['title'], thumbnail_url, web_series_data['hd_link'], web_series_data['hd_price'], False, category)
    bot.send_message(message.chat.id, f"Web series '{web_series_data['title']}' added successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: message.text == 'add free movie' and str(message.from_user.id) == ADMIN_ID)
def handle_add_free_movie_thumbnail(message):
    """Handles the 'add free movie' command (admin)."""
    bot.send_message(message.chat.id, "Please send the free movie thumbnail.")
    bot.register_next_step_handler(message, get_free_movie_name)

def get_free_movie_name(message):
    """Gets the free movie name from the admin."""
    global free_movie_data
    free_movie_data = {'thumbnail': message.photo[0].file_id if message.photo else message.text} # Handle both photo and URL
    bot.send_message(message.chat.id, "Please enter the free movie name.")
    bot.register_next_step_handler(message, get_free_movie_hd_link)

def get_free_movie_hd_link(message):
    """Gets the HD link for the free movie from the admin."""
    global free_movie_data
    free_movie_data['title'] = message.text
    bot.send_message(message.chat.id, "Please enter the HD Quality free movie link.")
    bot.register_next_step_handler(message, get_free_movie_category)

def get_free_movie_category(message):
    """Gets the category of the free movie."""
    global free_movie_data
    free_movie_data['hd_link'] = message.text.strip()
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add('New Latest', 'Popular', 'Oldest')
    bot.send_message(message.chat.id, "Select the free movie category:", reply_markup=markup)
    bot.register_next_step_handler(message, save_free_movie)

def save_free_movie(message):
    """Saves the free movie data to the database."""
    global free_movie_data
    category = message.text.lower().replace(" ", "_")
    # Get file_id of the thumbnail
    file_info = bot.get_file(free_movie_data['thumbnail'])
    thumbnail_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
    add_movie_to_db(free_movie_data['title'], thumbnail_url, free_movie_data['hd_link'], None, None, None, None, None, True, category)
    bot.send_message(message.chat.id, f"Free movie '{free_movie_data['title']}' added successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: message.text == 'add free web series' and str(message.from_user.id) == ADMIN_ID)
def handle_add_free_web_series_thumbnail(message):
    """Handles the 'add free web series' command (admin)."""
    bot.send_message(message.chat.id, "Please send the free web series thumbnail.")
    bot.register_next_step_handler(message, get_free_web_series_name)

def get_free_web_series_name(message):
    """Gets the free web series name from the admin."""
    global free_web_series_data
    free_web_series_data = {'thumbnail': message.photo[0].file_id if message.photo else message.text} # Handle both photo and URL
    bot.send_message(message.chat.id, "Please enter the free web series name.")
    bot.register_next_step_handler(message, get_free_web_series_hd_link)

def get_free_web_series_hd_link(message):
    """Gets the HD link for the free web series from the admin."""
    global free_web_series_data
    free_web_series_data['title'] = message.text
    bot.send_message(message.chat.id, "Please enter the HD Quality free web series link.")
    bot.register_next_step_handler(message, get_free_web_series_category)

def get_free_web_series_category(message):
    """Gets the category of the free web series."""
    global free_web_series_data
    free_web_series_data['hd_link'] = message.text.strip()
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add('New Latest', 'Popular', 'Oldest')
    bot.send_message(message.chat.id, "Select the free web series category:", reply_markup=markup)
    bot.register_next_step_handler(message, save_free_web_series)

def save_free_web_series(message):
    """Saves the free web series data to the database."""
    global free_web_series_data
    category = message.text.lower().replace(" ", "_")
    # Get file_id of the thumbnail
    file_info = bot.get_file(free_web_series_data['thumbnail'])
    thumbnail_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
    add_web_series_to_db(free_web_series_data['title'], thumbnail_url, free_web_series_data['hd_link'], None, True, category)
    bot.send_message(message.chat.id, f"Free web series '{free_web_series_data['title']}' added successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: message.text.lower() == 'edit paid movie' and str(message.from_user.id) == ADMIN_ID)
def handle_edit_paid_movie(message):
    """Handles the 'edit paid movie' command (admin)."""
    bot.send_message(message.chat.id, "Please enter the name of the paid movie you want to edit or delete.")
    bot.register_next_step_handler(message, show_delete_paid_movie_button)

def show_delete_paid_movie_button(message):
    """Shows the delete button for paid movies."""
    movie_name = message.text
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add(f'Delete Paid Movie: {movie_name}')
    bot.send_message(message.chat.id, f"Do you want to delete '{movie_name}'?", reply_markup=markup)
    bot.register_next_step_handler(message, delete_paid_movie_confirmation)

def delete_paid_movie_confirmation(message):
    """Confirms and deletes the paid movie."""
    if message.text.startswith('Delete Paid Movie: '):
        movie_name = message.text[len('Delete Paid Movie: '):]
        delete_movie(movie_name)
        bot.send_message(message.chat.id, f"Paid movie '{movie_name}' deleted successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())
    else:
        bot.send_message(message.chat.id, "Deletion cancelled.", reply_markup=telebot.types.ReplyKeyboardRemove())

# Similar handlers for 'edit paid web series', 'edit free movie', 'edit free web series' and their delete confirmations

@bot.message_handler(func=lambda message: message.text.lower() == 'edit paid web series' and str(message.from_user.id) == ADMIN_ID)
def handle_edit_paid_web_series(message):
    bot.send_message(message.chat.id, "Please enter the name of the paid web series you want to delete.")
    bot.register_next_step_handler(message, show_delete_paid_web_series_button)

def show_delete_paid_web_series_button(message):
    web_series_name = message.text
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add(f'Delete Paid Web Series: {web_series_name}')
    bot.send_message(message.chat.id, f"Do you want to delete '{web_series_name}'?", reply_markup=markup)
    bot.register_next_step_handler(message, delete_paid_web_series_confirmation)

def delete_paid_web_series_confirmation(message):
    if message.text.startswith('Delete Paid Web Series: '):
        web_series_name = message.text[len('Delete Paid Web Series: '):]
        delete_web_series(web_series_name)
        bot.send_message(message.chat.id, f"Paid web series '{web_series_name}' deleted successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())
    else:
        bot.send_message(message.chat.id, "Deletion cancelled.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: message.text.lower() == 'edit free movie' and str(message.from_user.id) == ADMIN_ID)
def handle_edit_free_movie(message):
    bot.send_message(message.chat.id, "Please enter the name of the free movie you want to delete.")
    bot.register_next_step_handler(message, show_delete_free_movie_button)

def show_delete_free_movie_button(message):
    free_movie_name = message.text
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add(f'Delete Free Movie: {free_movie_name}')
    bot.send_message(message.chat.id, f"Do you want to delete '{free_movie_name}'?", reply_markup=markup)
    bot.register_next_step_handler(message, delete_free_movie_confirmation)

def delete_free_movie_confirmation(message):
    if message.text.startswith('Delete Free Movie: '):
        free_movie_name = message.text[len('Delete Free Movie: '):]
        delete_movie(free_movie_name)
        bot.send_message(message.chat.id, f"Free movie '{free_movie_name}' deleted successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())
    else:
        bot.send_message(message.chat.id, "Deletion cancelled.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: message.text.lower() == 'edit free web series' and str(message.from_user.id) == ADMIN_ID)
def handle_edit_free_web_series(message):
    bot.send_message(message.chat.id, "Please enter the name of the free web series you want to delete.")
    bot.register_next_step_handler(message, show_delete_free_web_series_button)

def show_delete_free_web_series_button(message):
    free_web_series_name = message.text
    markup = telebot.types.ReplyKeyboardMarkup(one_time_keyboard=True)
    markup.add(f'Delete Free Web Series: {free_web_series_name}')
    bot.send_message(message.chat.id, f"Do you want to delete '{free_web_series_name}'?", reply_markup=markup)
    bot.register_next_step_handler(message, delete_free_web_series_confirmation)

def delete_free_web_series_confirmation(message):
    if message.text.startswith('Delete Free Web Series: '):
        free_web_series_name = message.text[len('Delete Free Web Series: '):]
        delete_web_series(free_web_series_name)
        bot.send_message(message.chat.id, f"Free web series '{free_web_series_name}' deleted successfully.", reply_markup=telebot.types.ReplyKeyboardRemove())
    else:
        bot.send_message(message.chat.id, "Deletion cancelled.", reply_markup=telebot.types.ReplyKeyboardRemove())

@bot.message_handler(func=lambda message: True)
def handle_all_other_messages(message):
    """Handles all other messages by showing popular content."""
    popular_movies = get_popular_movies()
    popular_web_series = get_popular_web_series()
    bot.send_message(message.chat.id, "Here are some popular movies and web series:")
    send_movie_list(message.chat.id, popular_movies)
    send_web_series_list(message.chat.id, popular_web_series)

# --- Payment Handling (Conceptual - Requires Cashfree Integration) ---
# You'll need to implement the Cashfree API calls here
# This would likely involve handling inline keyboard callbacks after a user selects a quality

# --- Referral Tracking ---
# Implement logic to track referrals and notify users

# --- Main Loop ---
if __name__ == '__main__':
    create_tables() # Ensure tables exist
    bot.polling(none_stop=True)
