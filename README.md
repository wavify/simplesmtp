# simplesmtp

This is a module to easily create custom SMTP servers and clients - use SMTP as a first class protocol in Node.JS!

[![Build Status](https://secure.travis-ci.org/andris9/simplesmtp.png)](http://travis-ci.org/andris9/simplesmtp)

[Autogenerated docs](http://node.ee/smtpdoc/)

# SMTP Server

## Usage

Create a new SMTP server instance with

    var smtp = new simplesmtp.createServer([options]);
    
And start listening on selected port

    smtp.listen(25, [function(err){}]);
    
SMTP options can include the following:

  * **name** - the hostname of the server, will be used for informational messages
  * **debug** - if set to true, print out messages about the connection
  * **timeout** - client timeout in milliseconds, defaults to 60 000 (60 sec.)
  * **secureConnection** - start a server on secure connection
  * **SMTPBanner** - greeting banner that is sent to the client on connection
  * **requireAuthentication** - if set to true, require that the client must authenticate itself
  * **validateSender** - if set to true, emit `'validateSender'` with `email` and `callback` when the client enters `MAIL FROM:<address>`
  * **validateRecipients** - if set to true, emit `'validateRecipient'` with `email` and `callback` when the client enters `RCPT TO:<address>`
  * **maxSize** - maximum size of an e-mail in bytes (currently informational only)
  * **credentials** - TLS credentials (`{key:'', cert:'', ca:''}`) for the server
  * **authMethods** - allowed authentication methods, defaults to `["PLAIN", "LOGIN"]`
  * **disableEHLO** - if set to true, support HELO command only
  
## Example

    var simplesmtp = require("simplesmtp"),
        fs = require("fs");

    var smtp = new simplesmtp.createServer();
    smtp.listen(25);

    smtp.on("startData", function(envelope){
        console.log("Message from:", envelope.from);
        console.log("Message to:", envelope.to);
        envelope.saveStream = fs.createWriteStream("/tmp/message.txt");
    });
    
    smtp.on("data", function(envelope, chunk){
        envelope.saveStream.write(chunk);
    });
    
    smtp.on("dataReady", function(envelope, callback){
        envelope.saveStream.end();
        console.log("Incoming message saved to /tmp/message.txt");
        callback(null, "ABC1"); // ABC1 is the queue id to be advertised to the client
        // callback(new Error("That was clearly a spam!"));
    });


## Events

  * **startData** *(envelope)* - DATA stream is opened by the client (`envelope` is an object with `from`, `to`, `host` and `remoteAddress` properties)
  * **data** *(envelope, chunk)* - e-mail data chunk is passed from the client 
  * **dataReady** *(envelope, callback)* - client is finished passing e-mail data, `callback` returns the queue id to the client
  * **authorizeUser** *(envelope, username, password, callback)* - will be emitted if `requireAuthentication` option is set to true. `callback` has two parameters *(err, success)* where `success` is Boolean and should be true, if user is authenticated successfully
  * **validateSender** *(envelope, email, callback)* - will be emitted if `validateSender` option is set to true
  * **validateRecipient** *(envelope, email, callback)* - will be emitted it `validataRecipients` option is set to true
  
# SMTP Client

## Usage

SMTP client can be created with `simplesmptp.connect(port[,host][, options])`
where

  * **port** is the port to connect to
  * **host** is the hostname to connect to (defaults to "localhost")
  * **options** is an optional options object (see below)
  
### Connection options

The following connection options can be used with `simplesmtp.connect`:

  * **secureConnection** - use SSL
  * **name** - the name of the client server
  * **auth** - authentication object `{user:"...", pass:"..."}`
  * **ignoreTLS** - ignore server support for STARTTLS
  * **debug** - output client and server messages to console
  * **instanceId** - unique instance id for debugging (will be output console with the messages)

### Connection events

Once a connection is set up the following events can be listened to:

  * **'idle'** - the connection to the SMTP server has been successfully set up and the client is waiting for an envelope
  * **'message'** - the envelope is passed successfully to the server and a message stream can be started
  * **'ready'** `(success)` - the message was sent
  * **'rcptFailed'** `(addresses)` - not all recipients were accepted (invalid addresses are included as an array)
  * **'error'** `(err)` - An error occurred. The connection is closed and an 'end' event is emitted shortly
  * **'end'** - connection to the client is closed

### Sending an envelope

When an `'idle'` event is emitted, an envelope object can be sent to the server.
This includes a string `from` and an array of strings `to` property.

Envelope can be sent with `client.useEnvelope(envelope)`

    client.on("idle", function(){
        client.useEnvelope({
            from: "me@example.com",
            to: ["receiver1@example.com", "receiver2@example.com"]
        });
    });

The `to` part of the envelope includes **all** recipients from `To:`, `Cc:` and `Bcc:` fields.

If setting the envelope up fails, an error is emitted. If only some (not all)
recipients are not accepted, the mail can still be sent but an `rcptFailed`
event is emitted.

    client.on("rcptFailed", function(addresses){
        console.log("The following addresses were rejected: ", addresses);
    });

If the envelope is set up correctly a `'message'` event is emitted.

### Sending a message

When `'message'` event is emitted, it is possible to send mail. To do this
you can pipe directly a message source (for example an .eml file) to the client
or alternatively you can send the message with `client.write` calls (you also
need to call `client.end()` once the message is completed.

If you are piping a stream to the client, do not leave the `'end'` event out,
this is needed to complete the message sequence by the client. 

    client.on("message", function(){
        fs.createReadStream("test.eml").pipe(client);
    });

Once the message is delivered a `'ready'` event is emitted. The event has an
parameter which indicates if the message was transmitted( (true) or not (false).

    client.on("ready", function(success){
        if(success){
            console.log("The message was transmitted successfully");
        }
    });

### Error types

On errors

### About reusing the connection

You can reuse the same connection several times but you can't send a mail
through the same connection concurrently. So if you catch and `'idle'` event
lock the connection to a message process and unlock after `'ready'`.

On '`error'` events you should reschedule the message and on `'end'` events
you should recreate the connection.

### Closing the client

By default the client tries to keep the connection up. If you want to close it,
run `client.quit()` - this sends a `QUIT` command to the server and closes the
connection

    client.quit();

## License

**MIT**