---
title: Building a Simple Bot for Discord
---

My wife is a dedicated Clash of Clans player. From her, I recently learnt about [Discord](https://discordapp.com/), which is a "All-in-one voice and text chat for gamers". The gamers seem to have built thriving communities on the app, and my wife is an active member in one of them. She was having a chat with one of her fellow community members about getting a bot going which would auto-wish members on their birthdays. Sounded like a nice li'l weekend project for /me!

A bit of Googling revealed that there are a number of nice libraries to build a bot for Discord, and quite a few for Node.js, the weapon of my choice! I liked [discord.io](https://github.com/izy521/discord.io), "a small, single-file library for creating DiscordApp clients from Node.js". Coupled with a node scheduler library (I like [Agenda](https://github.com/rschmukler/agenda)), I figured the bot can be built quite easily.

## Before Starting to Code

You'll need to register yourself on Discord & login, after which they allow you to create a new "App" inside the [Developers Section](https://discordapp.com/developers/applications/me). Once done, you will be given the option to Create a Bot User for this app. Do that and you will have your bot. You need to note down two things here: the **Client ID** and the App Bot User **Token** (click to reveal them). Further, you may tick the Public Bot checkbox. You can also add a cool icon here, which will be the face of your bot.

Now that you have your app and your bot (a pretty boring one though, since it cannot do a thing yet!), it is a good idea to set up a new Discord Server where you can play around with the bot. The process is as easy as clicking on a + button after logging into Discord. Choose a cool name for your server (mine is called Botyard; ain't that geeky?)

Next, you need to add your bot to your server. Google says there are a number of ways to do this, but the simplest method for me was visiting https://discordapp.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=0 in a new tab while you are logged in and choosing your Server name in the following screen. Remember to replace CLIENT_ID with the one you had jotted down previously.

And that's about all the bootstrapping you'd need to do before you can start infusing life into your bot, viz. the coding.

## Coding

First things first: make a new directory to hold your code, create an empty `index.js` file (all our code will be written in this) and run `npm init` within it. After answering (or skipping) the stock questions, you can go on to install the required dependencies using `npm i -S moment agenda discord.io`

Now let's just put in the usual skeleton code to load the libraries and all in the `index.js` file (you have it opened in your favorite editor, right?):

```javascript
const moment = require('moment')

const Agenda = require('agenda')
const agenda = new Agenda({
  db: {address: process.env.MONGO_CONNECTION_STRING},
  processEvery: '30 minutes'
});

const Discord = require('discord.io');
const bot = new Discord.Client({
    autorun: true,
    token: process.env.BOT_TOKEN
});
```


