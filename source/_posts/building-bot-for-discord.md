---
title: Building a Simple Bot for Discord
date: 2017/05/06 00:00:00
categories:
- JavaScript
- Node
tags:
- Node JS
- Discord Bot
---

My wife is a dedicated Clash of Clans player. From her, I recently learnt about [Discord](https://discordapp.com/), which is a "All-in-one voice and text chat for gamers". The gamers seem to have built thriving communities on the app, and my wife is an active member in one of them. She was having a chat with one of her fellow community members about getting a bot going which would auto-wish members on their birthdays. Sounded like a nice li'l weekend project for /me!

A bit of Googling revealed that there are a number of nice libraries to build a bot for Discord, and quite a few for Node.js, the weapon of my choice! I liked [discord.io](https://github.com/izy521/discord.io), "a small, single-file library for creating DiscordApp clients from Node.js". Coupled with a node scheduler library (I like [Agenda](https://github.com/rschmukler/agenda)), I figured the bot can be built quite easily.

## Before Starting to Code

You'll need to register yourself on Discord & login, after which they allow you to create a new "App" inside the [Developers Section](https://discordapp.com/developers/applications/me). Once done, you will be given the option to Create a Bot User for this app. Do that and you will have your bot. You need to note down two things here: the **Client ID** and the App Bot User **Token** (click to reveal them). Further, you may tick the Public Bot checkbox. You can also add a cool icon here, which will be the face of your bot.

Now that you have your app and your bot (a pretty boring one though, since it cannot do a thing yet!), it is a good idea to set up a new Discord Server where you can play around with the bot. The process is as easy as clicking on a + button after logging into Discord. Choose a cool name for your server (mine is called Botyard; ain't that geeky?)

Next, you need to add your bot to your server. Google says there are a number of ways to do this, but the simplest method for me was visiting https://discordapp.com/oauth2/authorize?client_id=CLIENT_ID&scope=bot&permissions=0 in a new tab while you are logged in and choosing your Server name in the following screen. Remember to replace CLIENT_ID with the one you had jotted down previously.

Finally, we need to have a place where we can store our data which is used by Agenda for persisting its schedules. A local running instance of MongoDB will be fine for testing. You'll need to note down the mongo connection string which is of the format `mongodb://<dbuser>:<dbpass>@<dbhost>:<dbport>`

And that's about all the bootstrapping you'd need to do before you can start infusing life into your bot, viz. the coding.

## Coding

First things first: make a new directory to hold your code, create an empty `index.js` file (all our code will be written in this) and run `npm init` within it. After answering (or skipping) the stock questions, you can go on to install the required dependencies using `npm i -S moment agenda discord.io`

Now let's just put in the usual skeleton code to load the libraries and all in the `index.js` file (you have it opened in your favorite editor, right?):

```javascript
const moment = require('moment')  // Dealing with dates? Don't leave home without this lib!

const Agenda = require('agenda')
const agenda = new Agenda({
  db: {address: process.env.MONGO_CONNECTION_STRING},
  processEvery: '30 minutes'	// Because we are doing birthdays here, this is kept much higher than the default
});

const Discord = require('discord.io');
const bot = new Discord.Client({
    autorun: true,
    token: process.env.BOT_TOKEN
});

agenda.on('ready', function() {
  agenda.start();
});

function graceful() {
  agenda.stop(function() {
    process.exit(0);
  });
}
process.on('SIGTERM', graceful);
process.on('SIGINT' , graceful);
```

Pretty self-explanatory, right? Note that, we will be passing our secrets (MONGO_CONNECTION_STRING & BOT_TOKEN) to the code using environmental variables at runtime. Also, using the `graceful()` function is the official Agenda way of ensuring that job processing is resumed post server shutdown/restarts.


Before writing any more code, let us take a step back and strategize a bit. That way, (and if you peruse the code comments), little to no explanation would be required when you see the actual code.

Here's what we would like our code to achieve:

* Define an Agenda job for sending a birthday wish to a particular user as a message from the Bot
* Setup the bot to listen to messages, and parse the ones in a pre-defined format as a "command" to get a user's handle and her birthday
* Use the above info to set up an Agenda schedule to wish the particular use on her birthday, and repeat it every year


The first and the third points can be handled easily using the following functions (Modularization is cool, and so is separation of concerns!):

``` javascript
let sendBirthdayMsg = function(user, channelID) {
    bot.sendMessage({
      to: channelID,
      message: `@${user} Happy Birthday to You!!!`
  });
}

agenda.define('send birthday wish', function(job, done) {
  let data = job.attrs.data; // we store whoseBirthday and channelID as properties of the job itself while scheduling it below
  sendBirthdayMsg(data.whoseBirthday, data.channelID)
  done()
});

let scheduleWish = function(whoseBirthday, whenBirthday, channelID) {
  agenda
  .create('send birthday wish', {whoseBirthday, channelID}) // refers to the job named 'send birthday wish' defined above
  .schedule(whenBirthday)
  .repeatEvery('1 year')
  .save()
}
```

Code for handling the second point is a bit longer, but, again, if you read the comments, should be pretty self-explanatory!
~~~ javascript
bot.on('message', function(user, userID, channelID, message, event) {
  // user, userID --> the user who sent the message 
  // (may contain a command for the bot, or may not!)
  // message --> the unadulterated message content; 
  // if   the message mentions another user (could be the bot), 
  // it   will include a substring in the format <@abcd124>, 
  // where   abcd1234 is the mentioned user's unique ID 
  // (Note that '<', '>' and '@' are all parts of the substring and not some placeholders!)
  // I'd recommend adding some console.log-s here to inspect the various parameters passed to this callback
  

  if (!message.includes(`<@${bot.id}>`)) return // ignore messages not aimed at the bot

  let botSays = ''  // this is what the bot would reply in response to a "command"

  // Clean Up: Remove the substring for the mentioned user (bot) and
  // strip all extra whitespaces
  let cleanedMsg = message.replace(`<@${bot.id}>`,'').replace(/ +(?= )/g,'').trim()   
  
  // Parse the message for command:
  // Format expected: <username> <birthday as MM-DD or DD/MM>
  let whoseBirthday = cleanedMsg.split(' ')[0]
  let whenBirthday = cleanedMsg.split(' ')[1]
  let whenBirthdayMoment = moment(whenBirthday, ['MM-DD','DD/MM'])  // p
  if ( !whenBirthdayMoment.isValid() ) {
    botSays = `@${user} Sorry Boss, I dint get what you just said!`
  } else {
  // check if b'day was on a past day
  if ( whenBirthdayMoment.isBefore(moment(), 'days') ) {
    whenBirthdayMoment.add(1, 'year') // no belated wishes; see you next year
  }
  scheduleWish(whoseBirthday, whenBirthdayMoment.toDate(), channelID)
    botSays = `@${user} OK Boss! I'll wish "${whoseBirthday}" every year on ${whenBirthdayMoment.format("Do MMMM")}!`
  }

  // bot completes doing its stuff & responds appropriately to the user issuing the command
  bot.sendMessage({	
    to: userID,
    message: botSays
  }) 
});
~~~


So, that's about it! You can view the completed code at [Github](https://github.com/sayanriju/boteswar).

To run the code, just clone the repo, `cd` into the directory, run `npm install` and finally issue the command: `MONGO_CONNECTION_STRING="mongodb://<dbuser>:<dbpass>@ds031952.mlab.com:31952/<dbname>" BOT_TOKEN="<your_token>" node index.js`. Obviously, replace the MONGO_CONNECTION_STRING and BOT_TOKEN values with your own.

Now you can go to your discord channel and chat around with the bot!

You'd notice that some extra stuff has been added in the github code. In particular, it has some random greeting texts instead of the hardcoded one discussed here. Also, not everybody is allowed to "command" the bot; a list of such privileged users who _can_ (I call them bosses) have been incorporated. You just need to add [your own User ID](https://www.reddit.com/r/discordapp/comments/40zgse/how_do_i_find_my_user_id/)  to the list.
