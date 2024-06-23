from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import CommandHandler, CallbackQueryHandler, MessageHandler, CallbackContext, ApplicationBuilder, filters
from gtts import gTTS
from moviepy.editor import VideoFileClip
import os
import logging
import speech_recognition as sr
from moviepy.video.tools.subtitles import SubtitlesClip

# Telegram bot token
TOKEN = 'YOUR_BOT_TOKEN'

# Set environment variable for ffmpeg
os.environ["IMAGEIO_FFMPEG_EXE"] = "/data/data/com.termux/files/usr/bin/ffmpeg"

# Creator ID
CREATOR_ID = '@Mahdi_00MR'

# Supported languages
LANGUAGES = {
    'fa': 'فارسی',
    'en': 'English'
}

# Set up logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)

# Start command handler
async def start(update: Update, context: CallbackContext) -> None:
    keyboard = [
        [InlineKeyboardButton("🇮🇷 / 🇦🇫 فارسی", callback_data='lang_fa')],
        [InlineKeyboardButton("🇺🇸 English", callback_data='lang_en')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text('لطفاً زبان خود را انتخاب کنید:', reply_markup=reply_markup)

# Set language handler
async def set_language(update: Update, context: CallbackContext) -> None:
    query = update.callback_query

    if query.data == 'lang_fa':
        context.user_data['lang'] = 'fa'
    elif query.data == 'lang_en':
        context.user_data['lang'] = 'en'

    await query.answer()
    await main_menu(update, context)

# Main menu handler
async def main_menu(update: Update, context: CallbackContext) -> None:
    lang = context.user_data.get('lang', 'fa')

    if lang == 'fa':
        keyboard = [
            [InlineKeyboardButton("تبدیل متن به صدا", callback_data='text_to_speech')],
            [InlineKeyboardButton("افزودن زیرنویس به ویدئو", callback_data='add_subtitle')]
        ]
        greeting_text = f'به منوی اصلی خوش آمدید\n\nسازنده: {CREATOR_ID}'
    else:
        keyboard = [
            [InlineKeyboardButton("Text to Speech", callback_data='text_to_speech')],
            [InlineKeyboardButton("Add Subtitle to Video", callback_data='add_subtitle')]
        ]
        greeting_text = f'Welcome to the main menu\n\nCreator: {CREATOR_ID}'

    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.callback_query.message.reply_text(greeting_text, reply_markup=reply_markup)

# Text to Speech handler
async def text_to_speech(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    lang = context.user_data.get('lang', 'fa')

    if lang == 'fa':
        prompt_text = 'لطفاً متن خود را برای تبدیل به صدا ارسال کنید:'
    else:
        prompt_text = 'Please send the text you want to convert to speech:'

    await query.answer()
    await query.message.reply_text(prompt_text)

# Handle text message for Text to Speech
async def handle_text_message(update: Update, context: CallbackContext) -> None:
    text = update.message.text
    lang = context.user_data.get('lang', 'fa')

    tts = gTTS(text, lang=lang)
    if not os.path.exists('./downloads'):
        os.makedirs('./downloads')
    file_path = './downloads/speech.mp3'
    tts.save(file_path)
    with open(file_path, 'rb') as audio_file:
        await update.message.reply_audio(audio_file)

# Add subtitle to video handler
async def add_subtitle(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    lang = context.user_data.get('lang', 'fa')

    if lang == 'fa':
        prompt_text = 'لطفاً ویدئوی خود را برای افزودن زیرنویس ارسال کنید:'
    else:
        prompt_text = 'Please send the video you want to add subtitles to:'

    await query.answer()
    await query.message.reply_text(prompt_text)

# Handle video message for adding subtitle
async def handle_video_message(update: Update, context: CallbackContext) -> None:
    video = await update.message.video.get_file()
    lang = context.user_data.get('lang', 'fa')  # Ensure lang is set

    if not os.path.exists('./downloads'):
        os.makedirs('./downloads')
    video_path = f'./downloads/{video.file_path.split("/")[-1]}'
    await video.download(video_path)

    # Extract audio from video
    audio_path = video_path.replace(".mp4", ".wav")
    video_clip = VideoFileClip(video_path)
    video_clip.audio.write_audiofile(audio_path)

    # Speech recognition
    recognizer = sr.Recognizer()
    with sr.AudioFile(audio_path) as source:
        audio_data = recognizer.record(source)
        try:
            text = recognizer.recognize_google(audio_data, language='fa-IR' if lang == 'fa' else 'en-US')
        except sr.UnknownValueError:
            await update.message.reply_text("Couldn't extract text from the video.")
            return
        except sr.RequestError as e:
            await update.message.reply_text(f"Speech recognition service error: {e}")
            return

    # Process subtitle addition (You can define your subtitle logic here)
    await update.message.reply_text(f"Extracted text: {text}")

# Error handler
async def error_handler(update: Update, context: CallbackContext) -> None:
    logging.error(f'Update {update} caused error {context.error}')

# Define and run the bot
async def main() -> None:
    application = ApplicationBuilder().token(TOKEN).build()

    application.add_handler(CommandHandler('start', start))
    application.add_handler(CallbackQueryHandler(set_language))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text_message))
    application.add_handler(CallbackQueryHandler(main_menu, pattern='main_menu'))
    application.add_handler(CallbackQueryHandler(text_to_speech, pattern='text_to_speech'))
    application.add_handler(CallbackQueryHandler(add_subtitle, pattern='add_subtitle'))
    application.add_handler(MessageHandler(filters.VIDEO & ~filters.COMMAND, handle_video_message))
    application.add_error_handler(error_handler)

    await application.run_polling()

if __name__ == '__main__':
    import asyncio
    asyncio.run(main()
  
