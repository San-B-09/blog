---
title: "Building your First Telegram Bot"
date: 2022-02-26
slug: "first-telegram-bot"
description: "Dive deep into developing and deploying your very own Chatbot!"
summary: "Dive deep into developing and deploying your very own Chatbot!"
categories: ["projects"]
draft: false
---

Chatbots are often touted as a revolution in the way users interact with technology and businesses. They have a fairly simple interface compared with traditional apps, as they only require users to chat, and the chatbots are supposed to understand and do whatever the user demands from them, at least in theory.

Chatbots boost operational efficiency and bring cost savings to businesses while offering convenience and added services to internal employees and external customers. They allow companies to easily resolve many types of customer queries and issues while reducing the need for human interaction. With chatbots, a business can scale, personalize, and be proactive all at the same time — which is an important differentiator.

There are a lot of options when it comes to where you can deploy your chatbot, and one of the most common uses are social media platforms, as most people use them on a regular basis. The same can be said of instant messaging apps, though with some limitations.

[Telegram](https://telegram.org/) is one of the more popular IM platforms today, as it allows you to store messages on the cloud instead of just your device and it boasts good multi-platform support, as you can have Telegram on Android, iOS, Windows, and just about any other platform that can support the web version. Building a chatbot on Telegram is fairly simple and requires few steps that take very little time to complete. The chatbot can be integrated in Telegram groups and channels, and it also works on its own.

Furthermore, we’ll be creating a base bot which could be extended according to your needs!

Our example will involve building a bot using Flask. To complete this, you will need [Python 3](https://www.python.org/downloads/) installed on your system as well as decent Python coding skills. Also, a good understanding of how apps work would be a good addition, but not a must, as we will be going through most of the stuff we present in detail.

And of course, you also need a Telegram account, which is absolutely free. You can sign up [here](https://telegram.org/).

# Bringing Your Telegram Bot to Life
To create a chatbot on Telegram, you need to contact the [BotFather](https://telegram.me/BotFather), which is essentially a bot used to create other bots.

The command you need is `/newbot` which leads to the following steps to create your bot:

![Telegram BotFather](/images/first-telegram-bot-telegram-botfather.png)

Your bot should have two attributes: a name and a username. The name will show up for your bot, while the username will be used for mentions and sharing.

After choosing your bot name and username — which must end with “bot” — you will get a message containing your access token, and you’ll obviously need to save your access token and username for later, as you will be needing them.

# Let’s get to the Code

We will be using Windows in this tutorial.

First, let’s create a virtual environment. It helps isolate your project’s requirements from your global Python environment.

```shell
python -m venv botEnv
```

Now we will have a `botEnv/` directory which will contain all the Python libraries we will be using. Go ahead and activate `botEnv` using the following command:

```shell
botEnv\Scripts\activate
```

The libraries we need for our bot are:

- [Flask](http://flask.pocoo.org/): A micro web framework built in Python.
- [Python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot): A Telegram wrapper in Python.
- [Decouple](https://pypi.org/project/python-decouple/): A Python library to store parameters in .env files.

You can install them in the virtual environment using pip command.

Now let’s browse our project directory.
```
Telegram-Bot-Base
    |--botEnv
    |--.env
    |--app.py
```

In the `.env` file we will need three variables:
```
API_KEY = "here goes your access token from BotFather"
BOT_USER_NAME = "the username you entered"
URL = "the hosting link that we will create later" 
```

Now let’s go back to our app.py and go through the code step by step:
```python
# import everything
from flask import Flask, request, abort
import telegram
from decouple import config

# Fetch variables from .env
API_KEY = config('API_KEY')
USER_NAME = config('BOT_USER_NAME')
URL = config('URL')

# initiate bot object with out API key
bot = telegram.Bot(token=API_KEY)
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
    # retrieve the message in JSON and transform it to Telegram object
    update = telegram.Update.de_json(request.get_json(force=True), bot)
    chat_id = update.message.chat.id
    msg_id = update.message.message_id

    if(update.message.text == None):
        return 'ok'

    # Telegram understands UTF-8, so encode text for its compatibility
    text = (update.message.text.encode('utf-8').decode()).lower()

    # the first time you chat with the bot AKA the welcoming message
    if '/start' in text:
        bot_welcome = """
        Hi, I'm the base bot.\nWanna know more about my source code ??\nHop onto this link: https://github.com/San-B-09/Telegram-Bot-Base
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,            
            reply_to_message_id=msg_id,
        )
    elif "hi" in text:
        bot_welcome = """
        Welcome to Base Bot. 
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,
            reply_to_message_id=msg_id,
        )
    elif "bye" in text:
        bot_welcome = """
        Byee!\nHave a productive time! 
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,
            reply_to_message_id=msg_id,
        )
    return 'ok'
```

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

Let’s take a look at the full version of app.py:
```python
# import everything
from flask import Flask, request, abort
import telegram
from decouple import config

# Fetch variables from .env
API_KEY = config('API_KEY')
USER_NAME = config('BOT_USER_NAME')
URL = config('URL')

# initiate bot object with out API key
bot = telegram.Bot(token=API_KEY)

# start the flask app
app = Flask(__name__)


@app.route('/{}'.format(API_KEY), methods=['POST'])
def respond():
    # retrieve the message in JSON and then transform it to Telegram object
    update = telegram.Update.de_json(request.get_json(force=True), bot)

    chat_id = update.message.chat.id
    msg_id = update.message.message_id

    if(update.message.text == None):
        return 'ok'

    # Telegram understands UTF-8, so encode text for unicode compatibility
    text = (update.message.text.encode('utf-8').decode()).lower()

    # the first time you chat with the bot AKA the welcoming message
    if '/start' in text:
        bot_welcome = """
        Hi, I'm the base bot.\nWanna know more about my source code ??\nHop onto this link: https://github.com/San-B-09/Telegram-Bot-Base
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,            
            reply_to_message_id=msg_id,
        )
    elif "hi" in text:
        bot_welcome = """
        Welcome to Base Bot. 
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,
            reply_to_message_id=msg_id,
        )
    elif "bye" in text:
        bot_welcome = """
        Byee!\nHave a productive time! 
        """
        bot.sendMessage(
            chat_id=chat_id, 
            text=bot_welcome,
            reply_to_message_id=msg_id,
        )

    return 'ok'


@app.route('/set_webhook', methods=['GET', 'POST'])
def set_webhook():
   s = bot.setWebhook('{URL}{HOOK}'.format(URL=URL, HOOK=API_KEY))
   if s:
       return "webhook setup ok"
   else:
       return "webhook setup failed"


@app.route('/')
def index():
   return '.'


if __name__ == '__main__':
   app.run(threaded=True)
```

That’s the last bit of code you will write in our tutorial. Now we can progress to the last step, launching our app on localhost and then on Heroku.

---

# Launch Our App on localhost

We need a couple of things before we make our app.

Telegram can’t communicate with the localhost directly and hence we’ll need a public URL to communicate with the bot.

Here, we’ll be using ngrok tunneling. ngrok is a free tool that allows us to tunnel from a public URL to our application running locally. You can download ngrok from [here](https://ngrok.com/download).

Activate the ngrok tunnel using following command:
```shell
ngrok http 5000
```

Copy the https URL tunneled to localhost to the URL variable of `.env` file.

![Ngrok Local Hosting](/images/first-telegram-bot-telegram-first-bot-localhost.png)

> Note: Make sure you append / at the end of URL in .env file.

Now run the app.py file. Then copy the ngrok URL and add to the end of the link `/setwebhook` so that the address will be something like `https://<ngrok code>.ngrok.io/setwebhook`. If you see `GET /set_webhook 200 OK` on ngrok’s terminal, that means you are ready to go!

---

# Launch Our App on Heroku

We need a couple of things before we make our app, including Git installed on your system.

Heroku can’t know what libraries your project uses, so we have to tell it using the `requirements.txt` file—a common problem is that you misspell requirements, so be careful—to generate the requirements file using pip:

```shell
pip freeze > requirements.txt
```

> Note: Make sure `requirements.txt` is created under appropriate environment.

Now you have your requirements file ready to go.

Now you need the `Procfile` which tells Heroku where our app starts, so create a `Procfile` file and add the following:
```shell
web: gunicorn app:app
```

> Note: Make sure you add `guicorn` to the  requirements.txt     separately for Heroku to install guicorn.

From your Heroku [dashboard](http://dashboard.heroku.com/apps/), create a new app. Once you do, it will direct you to the Deploy page. Then, click on open app and copy the domain of the app which will be something like `https://appname.herokuapp.com/` and paste it in the URL variable inside `.env`.

Now, go back to the *Deploy* tab and proceed with the steps:

Log in to Heroku:
```shell
heroku login
```

Please note that this method sometimes gets stuck in `waiting for login`, if this happens to you, try to log in using:
```shell
heroku login -i
```

Initialize a Git repository in our directory:

```shell
git init
heroku git:remote -a {heroku-project-name}
```

Deploy the app:
```shell
git add .
git commit -m "heroku commit"
git push heroku master
```

At this point, you will see the building progress in your terminal. If everything went okay, you will see something like this:
```shell
remote: -----> Launching...
remote:        Released v6
remote:        https://project-name.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
```

Now go to the app page (the link of the domain you copied before) and add to the end of the link `/set_webhook` so that the address will be something like `https://appname.herokuapp.com/set_webhook`. If you see `webhook setup ok`, that means you are ready to go!


# Go talk to your BOT!

![First Message](/images/first-telegram-bot-first-telegram-message.gif)

Now you can have your bot work the way you want — go ahead, make you customizations and create the next big thing!

I hope you had fun building your first Telegram bot.

**Hop onto complete source code here.**

{{< github repo="San-B-09/Telegram-Bot-Base" >}}

---

# References
1. [Python Telegram Bot documentation](https://sanketbijawe.medium.com/building-your-first-telegram-bot-cca7490ef60e#:~:text=Python%20Telegram%20Bot%20documentation)
2. [Python-telegram-bot Repository](https://github.com/python-telegram-bot/python-telegram-bot)
3. [Deploying with Git on Heroku](https://devcenter.heroku.com/articles/git)