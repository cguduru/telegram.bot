---
title: "Build a Chatbot with R"
---
# Table of Contents

In this page you will learn how to build a Telegram Bot with R and the `telegram.bot` package, with the following sections:

- [Creating a Telegram Bot](#creating-a-telegram-bot)
- [Introduction to the Telegram Bot API](#introduction-to-the-telegram-bot-api)
- [The telegram.bot Package](#the-telegrambot-package)
- [Building an R Bot in 3 steps](#building-an-r-bot-in-3-steps)
- [Adding Functionalities](#adding-functionalities)

So, let's *get started!*

# Creating a Telegram Bot

First, you must have or [create a Telegram account](https://web.telegram.org). Second, you'll need to create a Telegram Bot in order to get an Access Token. You can do so by talking to [*@BotFather*](https://telegram.me/botfather) and following a [few simple steps](https://core.telegram.org/bots#6-botfather). Telegram bots can receive *messages* or *commands*. The former are simply text that you send as if you were sending a message to another person, while the latter are prefixed with a `/` character. To create a new bot, send the following command to *BotFather* as a chat (exactly as if you were talking to another person on Telegram):

```bash
/newbot
```

You should get a reply instantly that asks you to choose a name for your Bot. You have to send then the name you want for the bot, which can be anyone, for instance:

```bash
RTelegramBot
```

*BotFather* will now ask you to pick a username for your Bot. This username has to end in `bot`, and be globally unique. In this tutorial we'll indicate the Bot's username with `<your-bot-username>`, so you'll have to substitute your chosen username wherever relevant from now on. Send your chosen username to *BotFather*:

```bash
<your-bot-username>
```
    
After doing so, *BotFather* will send you a "Congratulations" message, which will include a token. The token should look something like this:

`123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`

For the rest of this tutorial, we'll indicate where you need to put your token by using `<your-bot-token>` or just `TOKEN`. Take note of the token, as you'll need it in the code that you are about to write.

# Introduction to the Telegram Bot API

You can control your Bot by sending HTTPS requests to Telegram. This means that the simplest way to interact with your Bot is through a web browser. By visiting different URLs, you send different commands to your Bot. The simplest command is one where you get information about your Bot. Visit the following URL in your browser (substituting the `TOKEN` that you got before):

`https://api.telegram.org/bot<your-bot-token>/getMe`

The first part of the URL indicates that you want to communicate with the Telegram API (`api.telegram.org`). You follow this with `/bot` to say that you want to send a command to your Bot, and immediately after you add your `TOKEN` to identify which bot you want to send the command to and to prove that you own it. Finally, you specify the command that you want to send (`/getMe`) which in this case just returns basic information about our Bot using JSON.

## Retrieving messages sent to your Bot

The simplest way to retrieve messages sent to your Bot is through the `getUpdates` call:

`https://api.telegram.org/bot<your-bot-token>/getUpdates`

If you this page you'll get a JSON response of all the new messages sent to your Bot. Try sending a message to your Bot and visit that URL.

## Sending a message from your Bot

The final API call that we'll try out in the browser is that used to send a message. To do this, you need the chat ID for the chat where you want to send the message. There are a bunch of different IDs in the JSON response from the `getUpdates` call, so make sure you get the right one. It's the id field which is inside the chat field. Once you have this ID, visit the following URL in your browser, substituting `<chat-id>` for your chat ID.

`https://api.telegram.org/bot<your-bot-token>/sendMessage?chat_id=<chat-id>&text=TestReply`

Once you've visited this URL, you should see a message from your Bot sent to your which says "TestReply".

# The telegram.bot Package

You could program with R some functions that send these HTTPS requests and processes its responses. Fortunately, there is a package that allows you to do that: `telegram.bot`. It uses `httr` and `jsonlite` packages to do such work. Additionally, it features a number of tools to make the development of Telegram bots with R easy and straightforward, providing an easy-to-use interface that takes some work off the programmer.

Thereby, the `telegram.bot` package consists of several `R6` classes, and the API is exposed via the `Bot` class. The methods names are equivalents of the methods described in the official [Telegram Bot API](https://core.telegram.org/bots/api). The exact *snake_case* method names are also available for your convenience. So for example `Bot$get_updates` is the same as `Bot$getUpdates`.

## Creating a Bot instance

To get a feeling for the API and how to use it with `telegram.bot`, we will reproduce the URL based example we just saw, done with R with this package.

First, create an instance of the `Bot` class, where `TOKEN` should be replaced by the API token you received from *@BotFather*:

```r
# install.packages("telegram.bot")
library(telegram.bot)

bot <- Bot(token = "TOKEN")
```

To check if your credentials are correct, call the [getMe](https://core.telegram.org/bots/api#getme) API method:

```r
print(bot$getMe())
```

**Note:** Bots can't initiate conversations with users. A user must either add them to a group or send them a message first. People can use `telegram.me/<your-bot-username>` links or username search to find your bot (searching for `@<your-bot-username>` in any of the Telegram clients).

## Getting and retrieving messages

You can get updates from your bot with the command:

```r
updates <- bot$getUpdates()
```

This will retrieve a `list` generated from the JSON response from the server. In order to send a response, you can do it so with the following command:

```r
chat_id <- "CHAT_ID" # you can retrieve it from bot$getUpdates() after sending a message to the bot
bot$sendMessage(chat_id = chat_id, text = "TestReply")
```

Note that all methods accept their equivalent `snake_case` syntax (e.g. `bot$get_me()` is equivalent to `bot$getMe()`).

## Develop a Telegram Bot with R

In order to build a bot that is continuously running and is able to respond to multiple input data formats, the `telegram.bot` package features several `R6` classes, but the two most important ones here are `Updater` and `Dispatcher`.

The `Updater` class continuously fetches new updates from Telegram and passes them on to the `Dispatcher` class.  If you create an `Updater` object, it will create a `Dispatcher`. You can then register handlers of different types in the `Dispatcher`, which will sort the updates fetched by the `Updater` according to the handlers you registered, and deliver them to a callback function that you defined. Every handler is an instance of any subclass of the `Handler` class.

Finally you can run the bot with its method `start_polling()`, which runs a loop that has the following structure:

1. Get updates through the `getUpdates` method using Long Polling.
2. Dispatch each update to the appropriate process function.
3. Send an answer if specified in those functions.

Below we explain how to build a bot with this structure.

# Building an R Bot in 3 steps

In this section we will explain how to build a Bot with R and `telegram.bot` following the next steps:

1. [Creating the `Updater` object](#1-creating-the-updater-object)
2. [The first function](#2-the-first-function)
3. [Starting the Bot](#3-starting-the-bot)

## 1. Creating the Updater object

First, you first must create an `Updater` object. Replace `TOKEN` with your Telegram Bot's API Access Token, which looks something like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`.

```r
updater <- Updater(token = "TOKEN")
```

**Recommendation:** Following [Hadley's API
guidelines](http://github.com/hadley/httr/blob/master/vignettes/api-packages.Rmd#appendix-api-key-best-practices)
it's unsafe to type the `TOKEN` just in the R script. It's better to use
environment variables set in `.Renviron` file.

So let's say you have named your bot `RTelegramBot` (it's the first question
you answered to the *BotFather* when creating it); you can open the `.Renviron` file with the R commands:

```r
file.edit(path.expand(file.path("~", ".Renviron")))
```

And put the following line with
your `TOKEN` in your `.Renviron`:

```r
R_TELEGRAM_BOT_RTelegramBot=TOKEN
```
If you follow the suggested `R_TELEGRAM_BOT_` prefix convention you'll be able
to use the `bot_token` function (otherwise you'll have to get
these variable from `Sys.getenv`).

After you've finished these steps **restart R** in order to have
working environment variables. You can then create the `Updater` object as:

```r
updater <- Updater(token = bot_token("RTelegramBot"))
```

## 2. The first function

Now, you can define a function that should process a specific type of update:

```r
start <- function(bot, update){
  bot$sendMessage(chat_id = update$message$chat_id,
                  text = sprintf("Hello %s!", update$message$from$first_name))
}
```

The goal is to have this function called every time the Bot receives a Telegram message that contains the `/start` command.
To accomplish that, you can use a `CommandHandler` (one of the provided `Handler` sub-classes) and register it in the `updaters`'s `dispatcher` (which is done with the `+` operator):

```r
start_handler <- CommandHandler("start", start)
updater <- updater + start_handler
```

## 3. Starting the Bot

And that's all you need. To start the bot, run:

```r
updater$start_polling()
```

Give it a try! Start a chat with your bot and issue the `/start` command - if all went right, it will reply.

# Adding Functionalities

We have already built a Telegram Bot with R. However, it can now only answer to the `/start` command, so now we are going to add a couple of functionalities, including:

- [Text responses](#text-responses)
- [Commands with arguments](#commands-with-arguments)
- [Unknown command handling](#unknown-command-handling)
- [Stopping the Bot](#stopping-the-bot)
- [Advanced Filters](#advanced-filters)

## Text responses

Let's add another handler that listens for regular messages.
Use the `MessageHandler`, another `Handler` subclass, to echo to all text messages:

```r
echo <- function(bot, update){
	bot$sendMessage(chat_id = update$message$chat_id, text = update$message$text)
}

updater <- updater + MessageHandler(echo, MessageFilters$text)
```

From now on, your bot should echo all non-command messages it receives.

**Note:** As soon as you add new handlers to the `updaters`'s `dispatcher` (done with the `+` operator), they are in effect.

**Note:** The `MessageFilters` object contains a number of functions that filter incoming messages for text, images, status updates and more.
Any message that returns `TRUE` for at least one of the filters passed to `MessageHandler` will be accepted.
You can also write your own filters if you want.

## Commands with arguments

Let's add some actual functionality to your bot. We want to implement a `/caps` command that will take some text as an argument and reply to it in CAPS.
To make things easy, you can receive the arguments (as a `vector`, split on spaces) that were passed to a command in the callback function:

```r
caps <- function(bot, update, args){
  if (length(args > 0L)){
   	text_caps <- toupper(paste(args, collapse = " "))
   	bot$sendMessage(chat_id = update$message$chat_id,
   	                text = text_caps) 
  }
}

updater <- updater + CommandHandler("caps", caps, pass_args = TRUE)
```

**Note:** Take a look at the `pass_args = TRUE` in the `CommandHandler` initiation.
This is required to let the handler know that you want it to pass the list of command arguments to the callback.
All handler classes have keyword arguments like this. Some are the same among all handlers, some are specific to the handler class.
If you use a new type of handler for the first time, look it up in the docs and see if one of them is useful to you.

## Unknown command handling

Not bad! However, some confused users might try to send commands to the bot that it doesn't understand, so you can use a `MessageHandler` with a `command` filter to reply to all commands that were not recognized by the previous handlers.

```r
unknown <- function(bot, update){
	bot$sendMessage(chat_id = update$message$chat_id,
                        text = "Sorry, I didn't understand that command.")
}

updater <- updater + MessageHandler(unknown, MessageFilters$command)
```

## Stopping the Bot

If you're done playing around, you can stop the Bot either by using the the `interrupt R` command in the session menu (in *RStudio* you can press the `STOP` button) or by calling the `updater$stop_polling()` method. Below we will define a command that uses this method:

```r
# Example of a 'kill' command
kill <- function(bot, update){
  bot$sendMessage(chat_id = update$message$chat_id,
                  text = "Bye!")
  # Clean 'kill' update
  bot$getUpdates(offset = update$update_id + 1L)
  # Stop the updater polling
  updater$stop_polling()
}

updater <<- updater + CommandHandler("kill", kill)
```

Now you can send the command `/kill` from Telegram to stop the Bot. However, in a production environment it wouldn't be recommendable to leave this command as it is now, as anyone could stop the bot. To solve this, you can create a customized filter in order to make this command available only for a certain `user_id`, for instance. This is explained in the next section.

**Note:** With the [*superassignment* operator `<<-`](https://stat.ethz.ch/pipermail/r-help/2011-April/275905.html) we assign the `updater` in the enclosing environment so to call it from inside the `kill` function.

## Advanced Filters

It is also possible to write your own filters used with `MessageHandler` and `CommandHandler`. In essence, a filter is simply a function that receives a `Message` instance and returns either `TRUE` or `FALSE`. If a filter evaluates to `TRUE`, the message will be handled. 

This page describes advanced use cases for the filters used with `MessageHandler` or `CommandHandler`.

### Combining filters

When using `MessageHandler` it is sometimes useful to have more than one filter. This can be done using so called binary operators. Using `telegram.bot`, you can operate `BaseFilter` instances with `&`, `|` and `!` meaning AND, OR and NOT respectively.

#### Message is either video, photo, or document (generic file)

```r
handler <- MessageHandler(callback, MessageFilters$video | MessageFilters$photo | MessageFilters$document)
```

#### Message is a forwarded photo

```r
handler <- MessageHandler(callback, MessageFilters$forwarded & MessageFilters$photo)
```

#### Message is a photo and it's not forwarded

```r
handler <- MessageHandler(callback, MessageFilters$photo & (! MessageFilters$forwarded))
```

### Custom filters

It is also possible to write your own filters used with `MessageHandler` and `CommandHandler`. In essence, a filter is simply a function that receives a `Message` instance and returns either `TRUE` or `FALSE`. This function has to be implemented in a new class that inherits from `BaseFilter`, which allows it to be combined with other filters. If a filter evaluates to `TRUE`, the message will be handled. 

#### Restricting users

Say that for the `kill` example we saw previously, we would like to filter that command so to make it accessible only for a specific `USER_ID`. Thereby, you could add a filter:

```r
filter_user <- function(message) message$from_user  == "USER_ID"
```

You can make the function an instance of `BaseFilter` either with its generator:

```r
filter_user <- BaseFilter(filter = filter_user)
```

Or by coercing it with `as.BaseFilter`:

```r
filter_user <- as.BaseFilter(function(message) message$from_user  == "USER_ID")
```

Remember that to make it work, your filter must be a `function` that takes a `message` as input and returns a boolean: `TRUE` if the message should be handled, `FALSE` otherwise.

Now, you could update the handler with this filter:

```r
kill_handler <- CommandHandler("kill", kill, filter_user)
```

#### Text or command filter

Filters can also be added to the `MessageFilters` object. Within it, we can see that `MessageFilters$text` and `MessageFilters$command` are mutually exclusive, so we could add a filter for messages that can be either one of them. This would result as:

```r
MessageFilters$text_or_command <- BaseFilter(function(message) !is.null(message$text))
```

The class can of cause be named however you want, the only important things are:

- The class has to inherit from `BaseFilter`.
- It has to implement a `filter` method.
- The filter must be a `function` that takes a `message` as input and returns a boolean.

The filter can then be used as:

```r
handler <- MessageHandler(callback, MessageFilters$text_or_command)
```

That's it for now! With this tutorial you may have the first guidelines to develop your R bot.

# Want more?

If you want to learn more about Telegram Bots with R, you can look at these resources:
- Package `telegram.bot` [GitHub Repo](https://github.com/ebeneditos/telegram.bot) or its [Wiki](https://github.com/ebeneditos/telegram.bot/wiki) to look at all methods and features available.
- You can also check Telegram's documentation [Bots: An introduction for developers](http://core.telegram.org/bots) and [Telegram Bot API](http://core.telegram.org/bots/api) to familiarize with the API.