# Simple-filer

This library allows you to send files(even bigger than your memory allows) between 2 browsers(Chrome only at this moment). The file data go directly from browserA to browserB using the data channel of WebRTC as the underlying transport layer.

# Install
```bash
npm install simple-filer
```
# Prerequisites
* Chrome browser
* WebServer running on TLS with WebSocket support

# Usage
## server side
WebRTC needs a signaling server to exchange some meta data between peers before peers can talk to each other directly.
The regular approach is to use WebSocket. The following snippet uses nodeJS as an example:
```javascript
var users = {} // online users with key as userID, value as socket
const WebSocket = require('ws');
const wss = new WebSocket.Server({
  server: httpsServer
});

wss.on('connection', function(ws){
  ws.on('message', msg => {
    var msgObj = JSON.parse(msg);

    switch (msgObj.msgType) {
      case "signaling":
        var targetClient = users[msgObj.to];
        targetClient.send(msg) // forward the signaling data to the specified user
        break;
      default: console.log('Oops. unknown msg: ', msgObj)
    }
  });
})
```
All you need to do is parse the client message `{msgType: 'signaling', from: userID, to: userID, signalingData: data}`, and forward it to the specified user.

## client side
```javascript
var ws = new WebSocket('wss://127.0.0.1:8443');

var Filer = require('simple-filer')
var filer = new Filer({myID: 123, signalingChannel: ws})

ws.onmessage = msg => {
  var msgObj = JSON.parse(msg.data);
  switch (msgObj.msgType) {
    case "signaling":
      filer.handleSignaling(msgObj);
      break;
  }
};

filer.on('task', function(task) { // when you are about to send/receive a file, newTask event is fired
  console.log('new task: ', task); // this is where you can add task info on the webpage
});

filer.on('progress', function({fileID, progress, fileName, fileURL}){ // file transfer progress event
  console.log('file transfer progress: ', fileName, ': ', progress)
});

filer.on('status', function({fileID, status}){ // file transfer status: pending/sending/receiving/done/removed
  console.log('new status for ', fileID, ': ', status)
});
```
## Example app
Inside the example folder, there is a full-featured example app.
```bash
cd node_modules/simple-filer/example
npm install
npm run start
```
Open Chrome browser, go to `https://127.0.0.1:8443`(It's httpS). The first time you open this address, Chrome would show you a 'not secure' message.
That's because WebRTC requires WebServer to run on TLS. I use a self-signed certificate.

If you want to run this app in your local network, edit the `example/public/javascripts/bundle.js`, search for `var ws = new WebSocket('wss:`, change the IP address, then restart the app.
This example app works pretty well in local network(small office, home), because the data goes directly between two browsers, it's even faster than data copy using thumb-drive.
You can also open the `example/public/javascripts/src/demo.js` to know how it works. Here is a screenshot of this app:

![running demo](https://media.worksphere.cn/repo/simple-filer/demo-640.gif)


## API
```javascript
var filer = new Filer({myID: 123, signalingChannel: ws, webrtcConfig: configObject})
```
Create a new filer object(you should only create one such object in your web app).

`myID` is current user's ID provided by your application. `signalingChannel` is what WebRTC needs to exchange meta data between peers. Usually, you can pass a connected WebSocket client.
`webrtcConfig` is optional, passed to the underlying SimplePeer(an excellent WebRTC library) constructor. This argument should be like:
```
{iceServers: [{url: 'stun:stun.l.google.com:19302'}, {url: 'turn:SERVERIP:PORT', credential: 'secret', username: 'username'}, ...]}
```
SimplePeer already provides a default value for this argument, so you don't need to provide one.

```
filer.handleSignaling(message)
```
Call this method when your WebSocket client has received a signaling message. The signaling message must have the following properties:
* `msgType` - its value must be 'signaling'
* `from` - message sender's userID
* `to` - message receiver's userID
* `signalingData` - signaling data generated by underlying WebRTC

You don't need to compose this message yourself, you just call `filer.send())`, everything is done for you.
```
filer.send(receiverUID, file)
```
* `receiverUID` - the targeting user you want to send the file, UID is provided by your application
* `file` - the html file object.
After calling `filer.send()`, a new task is created for both sender and receiver.
```
filer.removeTask(taskID)
```
When the file is received and saved successfully, you can choose to remove it by calling `removeTask()`.

## Events
```
filer.on('task', function(taskData){})
```
Fired when you call `filer.send()` or when file receiver received a message about the incoming file.
This is the moment you could add a html table row with the taskData.
```
filer.on('progress, function({fileID, progress, fileName, fileURL}){})
```
Fired when part of the file has been sent/received. You can use this event to show a progress bar.
When the `progress` value hits 1 at receiving side, you can generate a link with `fileName` and `fileURL`.

```
filer.on('status', function({fileID, status}){})
```
Fired when status changed from one value to another. There are 5 status during a file transfer:
* `pending` - wait for P2P connection to be established
* `sending` - file is sending
* `receiving` - file is receiving
* `done` - file is done sending or receiving
* `removed` - `filer.removeTask()` is called, all the underlying data is removed

# Built with

* [Simple-peer](https://github.com/feross/simple-peer) - Simple WebRTC video/voice and data channels.

# Caveats
* This library is far from production ready, but it works well in small office, home.
* Although you can send many files to many peers at the same time, it's recommended against that.
JavaScript is not very efficient at handling binary data.

# Issues
When a file is in receiving, removing it might cause an error(refer to #1).

Besides, I have a personal chat web app([http://worksphere.cn/home](http://worksphere.cn/home), no registration needed).
I also created a dedicated discussion group in my chat app for this project. If you have more issues or suggestions, just go in there.


# License

This project is licensed under the [MIT License](/LICENSE).
