import telebot
import requests
from telebot import types  # For inline keyboard

# Replace with your actual bot token and API URL
TELEGRAM_BOT_TOKEN = "Example_TELEGRAM_BOT_TOKEN"
FLASK_API_URL = "Example_FLASK_API_URL"

# Initialize bot and session
bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)
session = requests.Session()

# In-memory user state (replace with persistent storage in production)
user_states = {}
userData = {}

def set_user_state(user_id, key, value):
    if user_id not in user_states:
        user_states[user_id] = {}
    user_states[user_id][key] = value

def get_user_state(user_id, key):
    return user_states.get(user_id, {}).get(key)

@bot.message_handler(commands=['start'])
def start(message):
    bot.reply_to(message, "به Wingo Markets خوش آمدید! لطفا ایمیل خود را وارد کنید:")
    set_user_state(message.chat.id, "step", "awaiting_email")

@bot.message_handler(func=lambda message: get_user_state(message.chat.id, "step") == "awaiting_email")
def handle_email(message):
    set_user_state(message.chat.id, "email", message.text)
    bot.reply_to(message, "متشکرم! حالا لطفا رمز عبور خود را وارد کنید:")
    set_user_state(message.chat.id, "step", "awaiting_password")

@bot.message_handler(func=lambda message: get_user_state(message.chat.id, "step") == "awaiting_password")
def handle_password(message):
    global userData

    try:
        # Retrieve email from user state
        email = get_user_state(message.chat.id, "email")
        if not email:
            bot.reply_to(message, "خطا: ایمیل یافت نشد. لطفا فرآیند را دوباره شروع کنید.")
            return

        password = message.text.strip()
        login_url = f"{FLASK_API_URL}/login"
        login_payload = {"email": email, "password": password}

        response = session.post(login_url, json=login_payload)
        if response.status_code == 200:
            response_data = response.json()
            token = response_data.get("accessToken")
            userData = response_data.get("client")

            print(f"First Name: '{userData.get('firstName', 'کاربر')}'")
            print(f"Last Name: '{userData.get('lastName', '')}'")

            if token:
                bot.reply_to(message, "ورود موفقیت‌آمیز بود! در حال دریافت اطلاعات...")
                fetch_referrals(message, token)
                set_user_state(message.chat.id, "step", "referrals_fetched")
            else:
                bot.reply_to(message, "ورود موفقیت‌آمیز بود، اما توکنی دریافت نشد.")
                set_user_state(message.chat.id, "step", "awaiting_password")

        elif response.status_code == 202:
            bot.reply_to(message, "نیاز به تایید دو مرحله‌ای (2FA) است. لطفا کد 2FA ارسال شده به ایمیل یا برنامه احراز هویت خود را وارد کنید.")
            set_user_state(message.chat.id, "step", "awaiting_2fa")
            @bot.message_handler(func=lambda msg: get_user_state(msg.chat.id, "step") == "awaiting_2fa")
            def handle_2fa_input(msg):
                twofa_code = msg.text.strip()
                set_user_state(msg.chat.id, "twofa_code", twofa_code)
                handle_2fa(msg)

        elif response.status_code == 401:
            bot.reply_to(message, "اعتبارنامه‌ها نامعتبر هستند. لطفا رمز عبور خود را دوباره وارد کنید:")
            set_user_state(message.chat.id, "step", "awaiting_password")

        else:
            bot.reply_to(message, "ورود ناموفق بود. لطفا اعتبارنامه‌های خود را بررسی کنید.")

    except requests.exceptions.RequestException as e:
        bot.reply_to(message, f"خطای شبکه رخ داده است: {str(e)}")

    except Exception as e:
        bot.reply_to(message, f"یک خطای غیرمنتظره رخ داده است: {str(e)}")


def handle_2fa(message):
    try:
        twofa_code = message.text.strip()
        #print(f"کد 2FA دریافت شد: {twofa_code}")
        twofa_url = f"{FLASK_API_URL}/2fa-check"
        twofa_payload = {"_auth_code": twofa_code, "_trusted": "on", "rememberMe": True}
        headers = {
            "Content-Type": "application/json",
        }
        response = session.post(twofa_url, json=twofa_payload, headers=headers)
        try:
            response_data = response.json()
        except ValueError:
            response_data = {"message": response.text}
        #print("pre200")
        if response.status_code == 200:
            print("after200")
            # Successful 2FA verification, retrieve the token
            token = response_data.get("accessToken")
            userData = response_data.get("client")
            #print("توکن: ", token)
            if token:
                fetch_referrals(message, token)  # Pass the token to fetch_referrals
            else:
                bot.reply_to(message, "تایید 2FA ناموفق بود، توکنی دریافت نشد.")
        else:
            # 2FA verification failed
            error_message = response_data.get("message", "تایید 2FA ناموفق بود. لطفا دوباره تلاش کنید.")
            bot.reply_to(message, f"خطا: {error_message}")

    except requests.exceptions.RequestException as e:
        bot.reply_to(message, f"خطای شبکه رخ داده است: {str(e)}")
        #print(f"خطای شبکه: {str(e)}")
    except Exception as e:
        bot.reply_to(message, f"یک خطای غیرمنتظره رخ داده است: {str(e)}")
        #print(f"خطای غیرمنتظره: {str(e)}")


def fetch_referrals(message, token):
    if not token:
        bot.reply_to(message, "توکن دسترسی وجود ندارد. لطفا دوباره وارد شوید.")
        return
    try:
        response = session.post(
            f"{FLASK_API_URL}/ib/referrals",
            headers={"Authorization": f"Bearer {token}"}
        )

        if response.status_code == 200:
            data = response.json()
            #print("داده‌ها: ", data)
            referrals = data.get("rows", {})

            # If 'rows' is a dictionary, convert it into a list of dictionaries
            if isinstance(referrals, dict):
                referrals = [{"id": k, "data": v} for k, v in referrals.items()]
                #print("تبدیل زیرمجموعه‌ها از دیکشنری به لیست:", referrals)

            # Debugging: print the referrals
            #print("زیرمجموعه‌ها:", referrals)

            if referrals:
                #bot.reply_to(message, "زیرمجموعه‌ها با موفقیت دریافت شدند.")
                filter_referrals(message, referrals)  # Pass the referrals list to filter_referrals
            else:
                bot.reply_to(message, "هیچ اطلاعاتی یافت نشد")
        else:
            error_message = response.json().get('message', 'خطای غیرمنتظره.')
            bot.reply_to(message, f"خطا {response.status_code}: {error_message}")

    except Exception as e:
        bot.reply_to(message, f"یک خطا رخ داده است: {str(e)}")


from datetime import datetime  # Add this import at the top of your file

def filter_referrals(message, referrals_list):
    try:
        # Initialize summary data
        summary_data = {
            'total_clients': 0,  # Total number of referrals
            'active_clients': 0,  # Clients with balance >= $100 and traded >= 5 lots in the last 30 days
            'total_balance': 0.0,  # Sum of all referral balances
            'current_month_lots': 0.0,  # Total lots traded by referrals in the current month
            'new_clients_current_month': 0,  # Clients registered in the current month
            'total_lots': 0.0  # Total lots traded by all referrals
        }

        # Get the current month and year
        current_month = datetime.now().month
        current_year = datetime.now().year

        # Iterate over each referral
        for referral in referrals_list:
            # Convert referral data to a dictionary
            referral_data = {item["key"]: item["value"] for item in referral.get("data", [])}

            # Extract relevant data
            balance = float(referral_data.get("balance", 0))
            lots_last_30_days = float(referral_data.get("lotsLast30Days", 0))
            current_month_lots = float(referral_data.get("currentMonthLots", 0))
            registration_date = referral_data.get("registrationDate", "")

            # Check if the client is new this month
            try:
                reg_date = datetime.strptime(registration_date, "%Y-%m-%d")
                if reg_date.month == current_month and reg_date.year == current_year:
                    summary_data['new_clients_current_month'] += 1
            except (ValueError, TypeError):
                pass  # Skip if registration date is invalid

            # Check if the client is active
            if balance >= 100 and lots_last_30_days >= 5:
                summary_data['active_clients'] += 1

            # Update totals
            summary_data['total_balance'] += balance
            summary_data['current_month_lots'] += current_month_lots
            summary_data['total_lots'] += float(referral_data.get("totalLots", 0))
            summary_data['total_clients'] += 1

        # Store summary data in user state for later use
        set_user_state(message.chat.id, "summary_data", summary_data)

        # Create summary message
        summary_message = (
            "خلاصه وضعیت زیرمجموعه‌ها:\n\n"
            f"تعداد کل زیرمجموعه‌ها: {summary_data['total_clients']}\n"
            f"تعداد مشتریان فعال: {summary_data['active_clients']}\n"
            f"مجموع بالانس زیرمجموعه‌ها: ${summary_data['total_balance']:,.0f}\n"
            f"حجم کل معاملات (لات): {summary_data['current_month_lots']:,.1f}\n\n"
            "آیا مایل به مشاهده جزئیات کامل هستید؟\n\n"
            "برای دیدن جزئیات کامل می‌توانید به پنل کاربری خود و بخش همکاری مراجعه کنید."
        )

        # Create buttons
        markup = types.InlineKeyboardMarkup()
        btn1 = types.InlineKeyboardButton(
            text="🌐 پنل کاربری",
            url="https://client.wingomarkets.com/ib/dashboard"
        )
        btn2 = types.InlineKeyboardButton(
            text="🏆 امتیاز جشنواره Wingo Wonderland",
            callback_data="festival_score"
        )
        markup.add(btn1, btn2)

        # Send summary message
        bot.send_message(
            message.chat.id,
            summary_message,
            reply_markup=markup
        )

    except Exception as e:
        #print(f"خطا در پردازش اطلاعات: {str(e)}")
        bot.reply_to(message, "خطا در پردازش اطلاعات. لطفا مجددا تلاش کنید.")

# Updated festival score handler
@bot.callback_query_handler(func=lambda call: call.data == "festival_score")
def handle_festival_score(call):
    global userData
    try:
        user_id = call.message.chat.id
        summary_data = get_user_state(user_id, "summary_data")

        if not summary_data:
            bot.answer_callback_query(call.id, "❌ ابتدا باید اطلاعات زیرمجموعه‌ها را دریافت کنید!")
            return

        # Minimum requirements check
        requirements = {
            'new_clients': (summary_data['new_clients_current_month'] >= 20, 20),
            'active_clients': (summary_data['active_clients'] >= 10, 10),
            'total_lots': (summary_data['current_month_lots'] >= 30, 30),
            'total_balance': (summary_data['total_balance'] >= 10000, 10000)
        }

        # Check if all minimum requirements are met
        meets_conditions = all(condition for condition, _ in requirements.values())

        # Always calculate scores
        score_new = summary_data['new_clients_current_month'] * 1
        score_active = summary_data['active_clients'] * 10
        score_lots = summary_data['current_month_lots'] * 5
        score_balance = (summary_data['total_balance'] // 5000) * 20
        total_score = score_new + score_active + score_lots + score_balance

        # Format numbers
        formatted_numbers = {
            'balance': f"{summary_data['total_balance']:,.0f}",
            'lots': f"{summary_data['current_month_lots']:,.1f}",
            'new_clients': summary_data['new_clients_current_month'],
            'active_clients': summary_data['active_clients']
        }

        # Create requirements status message
        requirements_status = (
            "شرایط حداقلی برای جدول‌بندی:\n"
            f"▫️ حداقل ۲۰ مشتری جدید: {formatted_numbers['new_clients']}/20\n"
            f"▫️ حداقل ۱۰ مشتری فعال: {formatted_numbers['active_clients']}/10\n"
            f"▫️ حداقل ۳۰ لات معامله: {formatted_numbers['lots']}/30\n"
            f"▫️ حداقل $۱۰٬۰۰۰ بالانس: ${formatted_numbers['balance']}/$10,000\n\n"
        )

        # Reward configuration
        rewards = {
            12500: "سفر دبی ✈️",
            5000: "مک بوک ایر 💻",
            2500: "آیپد مینی 📱",
            1600: "آیپد 📱",
            800: "اپل واچ ⌚️",
            400: "ایرپاد 🎧",
            200: "پک هدیه وینگو 🎁"
        }

        # Determine eligible rewards if conditions are met
        eligible_rewards = []
        if meets_conditions:
            for threshold in sorted(rewards.keys(), reverse=True):
                if total_score >= threshold:
                    eligible_rewards.append(rewards[threshold])

        # Build reward message
        if meets_conditions:
            reward_message = (
                "🎉 شما واجد شرایط شرکت در جشنواره هستید!\n\n"
                "جوایز قابل دستیابی:\n" +
                ("\n".join([f"- {reward}" for reward in eligible_rewards])
                 if eligible_rewards
                 else "هنوز به اولین جایزه نرسیده‌اید ❌")
            )
        else:
            reward_message = "❌ برای دریافت جوایز باید تمام شرایط حداقلی را احراز کنید"

        # Build full message
        full_message = (
            f"سلام {userData.get('firstName', 'کاربر')} {userData.get('lastName', '')} عزیز 👋\n\n"
            f"{requirements_status}"
            f"🔸 مجموع امتیازات شما: {total_score}\n\n"
            f"{reward_message}"
        )

        bot.answer_callback_query(call.id)
        bot.send_message(
            call.message.chat.id,
            full_message,
            parse_mode='Markdown'
        )

    except Exception as e:
        #print(f"خطا: {str(e)}")
        bot.send_message(call.message.chat.id, "خطا در پردازش اطلاعات. لطفا مجددا تلاش کنید.")

if __name__ == "__main__":
    #print("ربات در حال اجرا است...")
    bot.infinity_polling()
