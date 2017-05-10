---
title: Realtime Chat with Express and Faye (Bayeux Protocol)
date: 2017/05/10 15:40:08
tags:
- Express
- Node
- JavaScript
- Realtime
- Chat
- Faye
- Bayeux
categories:
- JavaScript
- Node
---

If you Google around with keywords like "Realtime chat with Express", the one library that will definitely come up most often is __socket.io__. It is a GREAT library no doubt. However, if you check the tutorials, you will almost always find that they are giving a simplified version of using it. What I mean is that all their server side logic (viz. bootstrapping express, adding routing and sockets) is contained inside a single file. Again, that per se is perfectly ok for a tutorial, but when you try to extrapolate these teachings in your own workflow and/or a running code base, you might face certain small but irksome implementation hiccups. At least I did! I did manage to sort them out eventually (thanks to the collective wisdom that Google bestows upon us), but it still felt kinda hackish and unclean.

So, when one of the ongoing projects that the firm I'm associated with needed a simple realtime communication module, I set out to look for an alternative. Luckily, it turned out that such a solution does exists. Its a beautiful little library called [Faye](https://faye.jcoglan.com/) which is a "**publish-subscribe** messaging system based on the [Bayeux](https://docs.cometd.org/current/reference/index.html#_bayeux)  protocol". We will not go much into the subtle differences between socket.io and faye/bayeux (there _are_ technical differences, though cosmetically they may seem similar) in this post, but will rather put the latter to good use and build a simple little realtime chat interface, which would be minimally invasive on a standard Express app.

## Server Side

To begin with, we'll scaffold an Express app using the [official Express Application Generator](https://expressjs.com/en/starter/generator.html). I assume you already have it installed globally via npm.
~~~ javascript
express real-time-chat-example --ejs
cd real-time-chat-example 
npm install
~~~
We'll be using EJS as our templating engine here. That's only because EJS is the one I personally use; other engines should work perfectly fine as well.

Now let's just have Faye join the fray:

~~~ javascript
npm install --save faye
~~~
 Next, we add our code to bootstrap Faye, and what better place to do so than where all the Express bootstrapping happens, the entry point of your Express app, the `bin/www` file!
~~~ javascript
var app = require('../app');
var debug = require('debug')('faye-express:server');
var http = require('http');
var faye = require('faye');  // added by us
var bayeux = new faye.NodeAdapter({mount: '/faye', timeout: 45});  // added by us

// ..............

var server = http.createServer(app);
bayeux.attach(server); // added by us

// ...............

~~~
That's it! You can now run the express app as usual: `./bin/www` and visit http://localhost:3000 to see the familiar Welcome to Express page. The only difference here with a stock express app is that Faye has quietly started a central message server mounted at the path `/faye` on localhost.

Now all we need is a client or two to use the message server.

## Client Side

The Welcome to Express message you see comes from the `views/index.ejs` file which is rendered when you visit the / route of your app. This is where we'll add our Faye client. We'll modify the file, starting with the `<head>` part:

~~~ html
  <head>
    <title><%= title %></title>
    <link rel='stylesheet' href='/stylesheets/style.css' />
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
    <script type="text/javascript" src="http://localhost:3000/faye/client.js"></script>
    <script type="text/javascript" src="javascripts/script.js"></script>
  </head>
~~~

jQuery (we'll need it for rudimentary DOM manipulations) has been included from a CDN.
We'd have expected to add the Faye client lib from some CDN as well, but Faye is so nice that it offers its `client.js` file at a convenient location relative to your app's root url (http://localhost:3000) as soon as you've set it up on the server!
The `public/javascripts/script.js` file (which we need to create) shall contain our custom JS code.

Now, we'll add some more html to the `<body>` to constitute our rather spartan chat UI. If you need some spice, simply add your CSS in the `public/stylesheets/style.css` file.

~~~ html
  <body>	
    <form>
      <input type="text" id="new-message" placeholder="Your Message Here" />
      <input type="submit" id="submit-btn" value="Enter" />
    </form>
    <h4>Messages:</h4>
    <ul id="messages-list">
    </ul>
  </body>
~~~

Before writing the actual client side code, we should perhaps make a brief mention here about [PubSub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern), because that's the paradigm Faye/Bayeux is using.

In this model, we have _Clients_ which can _Publish_ and/or _Subscribe_ to _Channels_. Whenever one client publishes to a channel, all client-subscribers currently listening to that channel will be informed, in real time, of that publication, and they can act accordingly. The role of the message server here is that of a _Broker_ who manages the entire process transparently & behind the scenes, invisible for all practical purposes to the clients. It gives the impression that the clients are communicating amongst themselves directly.

Note that Faye/Bayeux which uses this PubSub paradigm is fundamentally different from the various implementations (or emulations) of the standard Sockets Protocol (e.g. sockets.io).



Armed with the above knowledge, its time to add our JS code to the `public/javascripts/script.js` file. The code should be pretty self-explanatory, esp.if you peruse the comments.

~~~ javascript
// Create the Faye client. 
// (All you need is the mount point for the Faye server you want to connect to)
var client = new Faye.Client('http://localhost:3000/faye')
 
$(document).ready(function () {

  $("#submit-btn").click(function (evt) {
    evt.preventDefault()
    var newMessage = $("#new-message").val()
    // We PUBLISH our new message to a CHANNEL (/messages)
    // which is dynamically "created" *in situ*, if required
    client.publish("/messages", {
      text: newMessage
    })
  })
    
  // Now we setup the client to SUBSCRIBE (listen in)
  // for messages coming into the same CHANNEL (/messages)
  // (Note that, the client which publishes the message itself is also subscribed)
  client.subscribe("/messages", function(newMessage) {
    $("#messages-list").append("<li>" + newMessage.text + "</li>")
  })
    
})
~~~

And that's all! Start the express app using `./bin/www` and open up two different browser windows/tabs each pointing to http://localhost:3000. Any message you input in one window will be seen immediately in the other.

So, in just a few lines of simple code, you have a working (albeit rudimentary) realtime chat system going! You can see the complete code at its [Github Repo](https://github.com/sayanriju/realtime-chat-express-faye). 

## Addendum: Saving Chats on the Server

While our realtime chat example works fine (except for the looks obviously), we would really like to save our chat history somewhere on the server. This can be done in two different ways.

The first option is to set up your server side Express routes the usual way. Have a POST route (say, `POST /message`) whose handler should receive the message content in its `req.body` and suitably save it to a database using usual code. On the client side, we can simply add an AJAX call to this route with the new message content as its payload. This AJAX call can occur in tandem with our existing code for faye publish. This method would work particularly well if you already have you Express routing system in place, and just need to add real time functionality without changing existing code or workflow.


The other method is to use Faye on the server as well with [Server Side Clients](https://faye.jcoglan.com/node/clients.html). It may sound like an oxymoron, but is extremely simple to implement. Just create a new JS file anywhere on the server, `require` it inside `app.js` (or for that matter, you can use the `app.js` file itself as well), and add the following code in it:

~~~ javascript
var Faye = require("faye")
var client = new Faye.Client('http://localhost:3000/faye')

client.subscribe('/messages', function (newMessage) {
  console.log("New Message: ", newMessage) // this will be shown in the server console
  // ... do stuff to actually save the new message to your DB
});
~~~
If the above snippet looks familiar, that's because it is! Faye's client code is *isomorphic*, i.e. same in essence irrespective of whether it appears on the server side or the client side. Also, you can add this code at any suitable place on the server with no friction with your existing codebase. 

