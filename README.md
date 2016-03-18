# ChatApp

<h3>Introduction</h3>

Writing a chat application with popular web applications stacks like PHP has traditionally been very hard. It involves polling the server for changes, keeping track of timestamps, and it’s a lot slower than it should be.

Sockets have traditionally been the solution around which most realtime chat systems are architected, providing a bi-directional communication channel between a client and a server.

This means that the server can push messages to clients. Whenever you write a chat message, the idea is that the server will get it and push it to all other connected clients.

<h3>The web framework</h3>
The first goal is to setup a simple HTML webpage that serves out a form and a list of messages. We’re going to use the Node.JS web framework express to this end. Make sure Node.JS is installed.

First let’s create a package.json manifest file that describes our project. I recommend you place it in a dedicated empty directory (I’ll call mine ChatApp).

<pre><code>{
  "name": "ChatApp",
  "version": "0.0.1",
  "description": "Basic Chat App with Node.js and Socket.io",
  "dependencies": {}
}</code></pre>

Now, in order to easily populate the dependencies with the things we need, we’ll use npm install --save

<pre><code>npm install --save express@4.10.2</code></pre>

Now that express is installed we can create an index.js file that will setup our application.

<pre><code>var app = require('express')();
var http = require('http').Server(app);

app.get('/', function(req, res){
  res.send('&lt;h1&gt;Hello world&lt;/h1&gt;');
});

http.listen(3000, function(){
  console.log('listening on *:3000');
});</code></pre>

This translates into the following:

1. Express initializes app to be a function handler that you can supply to an HTTP server (as seen in line 2).

2. We define a route handler / that gets called when we hit our website home.

3. We make the http server listen on port 3000.

<h3>Serving HTML</h3>
So far in index.js we’re calling res.send and pass it a HTML string. Our code would look very confusing if we just placed our entire application’s HTML there. Instead, we’re going to create a index.html file and serve it.

Let’s refactor our route handler to use sendFile instead:

app.get('/', function(req, res){

  res.sendFile(__dirname + '/index.html');
  
});

And populate index.html with the following:

<pre><code>&lt;!doctype html&gt;
&lt;html&gt;
  &lt;head&gt;
    &lt;title&gt;Socket.IO chat&lt;/title&gt;
    &lt;style&gt;
      * { margin: 0; padding: 0; box-sizing: border-box; }
      body { font: 13px Helvetica, Arial; }
      form { background: #000; padding: 3px; position: fixed; bottom: 0; width: 100%; }
      form input { border: 0; padding: 10px; width: 90%; margin-right: .5%; }
      form button { width: 9%; background: rgb(130, 224, 255); border: none; padding: 10px; }
      #messages { list-style-type: none; margin: 0; padding: 0; }
      #messages li { padding: 5px 10px; }
      #messages li:nth-child(odd) { background: #eee; }
    &lt;/style&gt;
  &lt;/head&gt;
  &lt;body&gt;
    &lt;ul id="messages"&gt;&lt;/ul&gt;
    &lt;form action=""&gt;
      &lt;input id="m" autocomplete="off" /&gt;&lt;button&gt;Send&lt;/button&gt;
    &lt;/form&gt;
  &lt;/body&gt;
&lt;/html&gt;</code></pre>

If you restart the process (by hitting Control+C and running node index again) and refresh the page.

<h3>Integrating Socket.IO</h3>
Socket.IO is composed of two parts:

1. A server that integrates with (or mounts on) the Node.JS HTTP Server: socket.io
2. A client library that loads on the browser side: socket.io-client

During development, socket.io serves the client automatically for us, as we’ll see, so for now we only have to install one module:
<pre><code>npm install --save socket.io</code></pre>

That will install the module and add the dependency to package.json. Now let’s edit index.js to add it:
<pre><code>var app = require('express')();
var http = require('http').Server(app);
var io = require('socket.io')(http);

app.get('/', function(req, res){
  res.sendFile('index.html');
});

io.on('connection', function(socket){
  console.log('a user connected');
});

http.listen(3000, function(){
  console.log('listening on *:3000');
});</code></pre>

Notice that I initialize a new instance of socket.io by passing the http (the HTTP server) object. Then I listen on the connection event for incoming sockets, and I log it to the console.

Now in index.html I add the following snippet before the <code>&lt;/body&gt;</code>:

<pre><code>&lt;script src="https://cdn.socket.io/socket.io-1.2.0.js"&gt;&lt;/script&gt;
&lt;script&gt;
  var socket = io();
&lt;/script&gt;</code></pre>

That’s all it takes to load the socket.io-client, which exposes a io global, and then connect.

Notice that I’m not specifying any URL when I call io(), since it defaults to trying to connect to the host that serves the page.

If you now reload the server and the website you should see the console print “a user connected”.
Try opening several tabs, and you’ll see several messages in the console.

Each socket also fires a special disconnect event:
<pre><code>io.on('connection', function(socket){
  console.log('a user connected');
  socket.on('disconnect', function(){
    console.log('user disconnected');
  });
});</code></pre>

<h3>Emitting events</h3>
The main idea behind Socket.IO is that you can send and receive any events you want, with any data you want. Any objects that can be encoded as JSON will do, and binary data is supported too.

Let’s make it so that when the user types in a message, the server gets it as a chat message event. The scripts section in index.html should now look as follows:
<pre><code>&lt;script src="https://cdn.socket.io/socket.io-1.2.0.js"&gt;&lt;/script&gt;
&lt;script src="http://code.jquery.com/jquery-1.11.1.js"&gt;&lt;/script&gt;
&lt;script&gt;
  var socket = io();
  $('form').submit(function(){
    socket.emit('chat message', $('#m').val());
    $('#m').val('');
    return false;
  });
&lt;/script&gt;</code></pre>

And in index.js we print out the chat message event:
<pre><code>io.on('connection', function(socket){
  socket.on('chat message', function(msg){
    console.log('message: ' + msg);
  });
});</code></pre>

<h3>Broadcasting</h3>
The next goal is for us to emit the event from the server to the rest of the users.

In order to send an event to everyone, Socket.IO gives us the io.emit:

<pre><code>io.emit('some event', { for: 'everyone' });</code></pre>

If you want to send a message to everyone except for a certain socket, we have the broadcast flag:
<pre><code>io.on('connection', function(socket){
  socket.broadcast.emit('hi');
});</code></pre>

In this case, for the sake of simplicity we’ll send the message to everyone, including the sender.
<pre><code>io.on('connection', function(socket){
  socket.on('chat message', function(msg){
    io.emit('chat message', msg);
  });
});</code></pre>

And on the client side when we capture a chat message event we’ll include it in the page. The total client-side JavaScript code now amounts to:
<pre><code>&lt;script&gt;
  var socket = io();
  $('form').submit(function(){
    socket.emit('chat message', $('#m').val());
    $('#m').val('');
    return false;
  });
  socket.on('chat message', function(msg){
    $('#messages').append($('&lt;li&gt;').text(msg));
  });
&lt;/script&gt;</code></pre>

And that completes our chat application, in about 20 lines of code!
