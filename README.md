# WingoTeleBot
**Overview**
The Wingo Markets Telegram Bot is a Python-based application designed to streamline user interaction with the Wingo Markets platform via Telegram. Built with the telebot library, this bot facilitates secure user authentication (including 2FA), retrieves referral data from the Wingo Markets API, and provides detailed summaries of referral performance, including client activity, balances, and trading volumes. It also calculates eligibility for the "Wingo Wonderland" festival rewards based on predefined criteria, offering an interactive experience with inline keyboards for seamless navigation.
**Features**
User Authentication: Handles email/password login and two-factor authentication (2FA) with the Wingo Markets API.
Referral Insights: Fetches and processes referral data, summarizing total clients, active clients, balances, and trading lots.
Festival Scoring: Computes user scores for the Wingo Wonderland festival, determining reward eligibility with dynamic updates.
Interactive Interface: Utilizes inline keyboards for easy access to the user dashboard and festival details.
Error Handling: Robust exception management for network issues, API errors, and invalid inputs.
**Prerequisites**
Python 3.8 or higher
Telegram account and bot token (via BotFather)
Access to the Wingo Markets API (replace FLASK_API_URL with your endpoint)
Dependencies: telebot, requests
