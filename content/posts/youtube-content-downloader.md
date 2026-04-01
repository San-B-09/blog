---
title: "YouTube Content Downloader - A Telegram Bot"
date: 2022-03-22
slug: "youtube-content-downloader"
description: "Build a Telegram bot that lets users instantly download YouTube videos or audio with a simple link."
summary: "Build a Telegram bot that lets users instantly download YouTube videos or audio with a simple link."
categories: ["projects"]
draft: false
---

An extension bot built on top of [Telegram Base Bot](https://blog.sanket0x.com/first-telegram-bot/).

YouTube being one of the largest video hosting platform, provides a simple way for people to store videos online and share them with others. Many among you might have also hopped onto YouTube for a video tutorial of building telegram bots.

In this tutorial blog, we’ll be creating a YouTube content downloader bot using [Python3](https://www.python.org/downloads/). A good understanding of how [Flask](https://flask.palletsprojects.com/en/2.0.x/) apps work would be a good addition, but not a must.

# Creating a New Telegram Bot
To create a chatbot on Telegram, you need to contact the [BotFather](https://telegram.me/BotFather), which is essentially a bot used to create other bots.

The command you need is `/newbot` which leads to the following steps to create your bot:

![Bot Father](/images/youtube-content-downloader-botfather.png)

Your bot should have two attributes: a name and a username. The name will show up for your bot, while the username will be used for mentions and sharing.

After choosing your bot name and username — which must end with “bot” — you will get a message containing your access token, and you’ll obviously need to save your access token and username for later, as you will be needing them.

# Let’s get to the Code

For environment set-up, please refer [Telegram Base Bot](https://blog.sanket0x.com/first-telegram-bot/).

The external libraries we need for our bot are:
- [Flask](https://sanketbijawe.medium.com/youtube-content-downloader-telegram-bot-1501ba2fd8f8#:~:text=our%20bot%20are%3A-,Flask,-%3A%20A%20micro%20web): A micro web framework built in Python.
- [Python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot): A Telegram wrapper in Python.
- [Decouple](https://pypi.org/project/python-decouple/): A Python library to store parameters in .env files.
- [youtube-dl](https://youtube-dl.org/): A Command-line program to download videos from YouTube.

You can install them in the virtual environment using pip command.

Now let’s browse our project directory.
```lua
Telegram-Bot-Base
    |--botEnv
    |--.env
    |--app.py
```

In the `.env` file we will need three variables:
```lua
API_KEY = "here goes your access token from BotFather"
BOT_USER_NAME = "the username you entered"
URL = "the hosting link that we will create later"
```

Now let’s go back to our app.py and go through the code step by step:

```python
# import everything
from flask import Flask, request
import telegram
from decouple import config
import youtube_dl as yt
import re
import os
import time

# Fetch variables from .env
API_KEY = config('API_KEY')
USER_NAME = config('BOT_USER_NAME')
URL = config('URL')

# initiate bot object with out API key
bot = telegram.Bot(token=API_KEY)

# initialize var to be used furthermore in code logic
YT_LINK = ""
YT_LINK_MSG_ID = ""
MAX_VIDEO_SIZE = 1100000
LAST_RECIEVED_MSG = ""
```

Now we have the bot object which will be used for any action we require the bot to perform.
```python
# start the flask app
app = Flask(__name__)
```

We also need to bind functions to specific routes. In other words, we need to tell Flask what to do when a specific address is called. More detailed info about Flask and routes can be found [here](http://flask.pocoo.org/docs/1.0/quickstart/).

In our example, the route function responds to a URL which is basically `/{token}`, and this is the URL Telegram will call to get responses for messages sent to the bot.

```python
@app.route('/{}'.format(API_KEY), methods=['POST'])
def respond():
    global YT_LINK
    global YT_LINK_MSG_ID
    global LAST_RECIEVED_MSG
    # retrieve the message in JSON and then transform it to Telegram object
    update = telegram.Update.de_json(request.get_json(force=True), bot)
    chat_id = update.message.chat.id
    msg_id = update.message.message_id
    if(update.message.text == None):
        return 'ok'
    # Telegram understands UTF-8, so encode text for unicode compatibility
    text = (update.message.text.encode('utf-8').decode())
    
    print("----------Recieved: {}".format(text))
    # the first time you chat with the bot AKA the welcoming message
    if '/start' == text:
        bot_welcome = """
        Hi, I'm the YouTube Downloader bot.\nSend in your YouTube video link to start download process.
        """
        bot.sendMessage(chat_id=chat_id, text=bot_welcome,
                        reply_to_message_id=msg_id)
    elif ('/video' == text or '/audio' == text):
        if (LAST_RECIEVED_MSG == text):
            return 'ok'
        LAST_RECIEVED_MSG = text
        if(YT_LINK == ""):
            bot.sendMessage(chat_id=chat_id, text="Youtube URL is not set. Kindly send youtube URL",
                            reply_to_message_id=YT_LINK_MSG_ID)
            return 'ok'
        
        bot.sendMessage(chat_id=chat_id, text="Thanks for using! Please wait for some time.")
        returnMsg, downloadedFileName = download_video(
            YT_LINK, chat_id, YT_LINK_MSG_ID, format=(text.split("/")[-1]))
        YT_LINK = ""
        YT_LINK_MSG_ID = ""
        if(returnMsg == 'ok'):
            bot.sendMessage(
                chat_id=chat_id, text="Sending...", reply_to_message_id=YT_LINK_MSG_ID)
        for file in os.listdir():
            if(downloadedFileName in file):
                bot.send_document(chat_id, open(file, 'rb'),
                                      reply_to_message_id=YT_LINK_MSG_ID, allow_sending_without_reply=True)
                break
           
        while(True):
            try:
                os.remove(file)
                break
            except:
                time.sleep(1)
                continue
        LAST_RECIEVED_MSG = ""
    else:
        regex = re.compile(r'youtube\.com|youtu\.be')
        if(regex.search(text)):
            YT_LINK = text
            YT_LINK_MSG_ID = msg_id
            buttons = [[telegram.KeyboardButton("/video")], [telegram.KeyboardButton("/audio")]]
            bot.sendMessage(chat_id=chat_id, text="Choose Downloading Format",
                            reply_to_message_id=msg_id,   reply_markup=telegram.ReplyKeyboardMarkup(buttons, one_time_keyboard=True))
        else:
            bot.sendMessage(
                chat_id=chat_id, text="Not an YouTube Link. Kindly send valid URL", reply_to_message_id=msg_id)
    return 'ok'
```

Here, we check if the received text is one of `/start`, `/video`, `/audio` or `/<YouTube Link>`. If the text is `/<YouTube Link>`, bot replies back with a button keyboard containing `/video` and `/audio` buttons. On selecting one of the option, bot calls `download_video()` with appropriate link and format as arguments. On successful download of the file (`video/audio`), `download_video()` returns `‘ok’` with downloaded file name. The file name is further used to send it back to the `chat_id(User)` and then delete the same from system to avoid any storage overheads.

`download_video()` is shown bellow:

```python
def download_video(link, chat_id, msg_id, format='video'):
    bot.sendMessage(chat_id=chat_id, text="Fetching Details...")
    try:
        with yt.YoutubeDL({}) as ydl:
            dictMeta = ydl.extract_info(link, download=False)
    except yt.utils.DownloadError:
        bot.sendMessage(chat_id=chat_id, text="Invalid URL!",
                        reply_to_message_id=msg_id)
        return "Invalid URL!", None
    if(format == 'video'):
        bot.sendMessage(chat_id=chat_id, text="Downloading Video...",
                        reply_to_message_id=msg_id)
        availableFormats = [format for format in dictMeta['formats'] if(
            format['filesize'] != None and format['filesize'] <= MAX_VIDEO_SIZE and format['ext'] == 'mp4')]
        if(len(availableFormats) == 0):
            bot.sendMessage(chat_id=chat_id, text="Video is Oversized!",
                            reply_to_message_id=msg_id)
            return "Video is Oversized", None
        sorted(availableFormats, key=lambda x: x['format_note'][:-1:])
        ydl_opts = {
            'format_id': availableFormats[-1]['format_id'],
            'outtmpl': './%(id)s.%(ext)s'
        }
    else:
        bot.sendMessage(chat_id=chat_id, text="Downloading Audio...",
                        reply_to_message_id=msg_id)
        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': './%(id)s.%(ext)s'
        }
    try:
        with yt.YoutubeDL(ydl_opts) as ydl:
            ydl.download([link])
    except:
        bot.sendMessage(chat_id=chat_id, text="Error in downloading. Please try again after some time!",
                        reply_to_message_id=msg_id)
    if('watch?v=' in link):
        downloadedFileName = link.split('watch?v=')[-1]
    else:
        downloadedFileName = link.split('/')[-1]
    return 'ok', downloadedFileName
```

Here, we’ll first verify if the link provided by user is valid or not by extracting the info using provided link. Furthermore, if the provided format is video, we extract the `format_id` from extracted info having extension as `.mp4` and with best video quality having file size bellow global `MAX_VIDEO_SIZE`. On contrary, if the chosen format is audio, we’ll simply choose best audio format. With `format_id/format`, `outtmpl` (Output File naming format), we’ll form a dictionary `ydl_opts` that is further sent to `YoutubeDL` object. Using the same object, we’ll call it’s `download` method and pass list of video links as it’s argument.

The intuitive way to make this function to work is that we will call it every second, so that it checks whether a new message has arrived, but we won’t be doing that. Instead, we will be using [Webhook](https://www.wikiwand.com/en/Webhook) which provides us a way of letting the bot call our server whenever a message is called, so that we don’t need to make our server suffer in a while loop waiting for a message to come.

So, we will make a function that we ourselves need to call to activate the Webhook of Telegram, basically telling Telegram to call a specific link when a new message arrives. We will call this function one time only, when we first create the bot. If you change the app link, then you will need to run this function again with the new link you have.

The route here can be anything; you’re the one who will call it:
```python
@app.route('/set_webhook', methods=['GET', 'POST'])
def set_webhook():
   s = bot.setWebhook('{URL}{HOOK}'.format(URL=URL, HOOK=API_KEY))
   if s:
       return "webhook setup ok"
   else:
       return "webhook setup failed"
```

Deployment for all telegram bots follow similar set of instructions for local host and Heroku. Hence, for deployment details refer [Telegram Base Bot](https://bijawesanket.medium.com/building-your-first-telegram-bot-cca7490ef60e).

# Go talk to your BOT!
![First Message](/images/youtube-content-downloader-first-message.gif)

**Hop onto complete source code here.**

{{< github repo="San-B-09/Telegram-YT-Downloader" >}}

# References
- [Python Telegram Bot documentation](https://python-telegram-bot.readthedocs.io/en/stable/index.html)
- [Python-telegram-bot Repository](https://github.com/python-telegram-bot/python-telegram-bot)
- [Deploying with Git on Heroku](https://devcenter.heroku.com/articles/git)
- [Youtube DL Repository](https://github.com/ytdl-org/youtube-dl)