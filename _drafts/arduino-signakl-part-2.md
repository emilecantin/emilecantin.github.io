---
layout: post
title: "Arduino and SignalK - Part 2: Getting your data on the server"
categories: web sysadmin iot javascript sailing
---

_This is part 2 of a multi-part series on outfitting my boat with some basic instrumentation, using open-source software and off-the-shelf electronics._

In the previous part of this series, we had a brief look at the hardware I had on hand for this project, as well as the basic software setup. Now, we'll have a look at what the documentation doesn't really cover: getting your own data into the SignalK server.

## Configuration

The previous part had us running the server via `bin/nmea-from-file`. If you look at that script, you'll see that it simply launches `bin/signalk-server` with a particular settings file.

Most of the example settings files (in the [`settings/`](https://github.com/SignalK/signalk-server-node/tree/master/settings) directory) are about converting NMEA data, which might be useful when you're integrating SystemK to an existing setup, but less so when you're starting from scratch.

Basically, a settings file has three main parts:

### `vessel`

That part is the easiest to understand. It consists of your boat name, UUID, and a few other things about your boat. I'm still not sure if any of it is actually used by the server, except for the UUID.

### `interfaces`

That part defines what interfaces are available to get our data from the server. The options look pretty much self-explanatory, but I haven't really messed with them.

### `pipedProviders`

This is the important one. You see, the SignalK server has two main sides: Consumers, which (obviously) consume the data, and do something with it (check out the [demo](http://signalk.org/demo.html) to see some examples in action), and Providers, which provide the data. You can see the list of available providers [here](https://github.com/SignalK/signalk-server-node/tree/master/providers).

As you can probably guess from the name of this config item, these providers are meant to be piped together. That is, the output of one is fed into the input of the following.

The `pipedProviders` section of the config is a list of the provider arrangements you want to have. A provider arrangement defines the flow of data; the first one should get it somewhere (either through running a command, receiving on a serial or TCP port, etc.), and the last one should output valid SignalK data.

## Arduino

As the title of this series says, I get my data from an Arduino. I don't want to run complicated calculations or data transformations on it, so I started from the serial port example, and I came up with this:

```c
#define ANALOG_QTY 4

int analogReading[ANALOG_QTY];
int analogReadingPrev[ANALOG_QTY];

void setup() {
  for(int i = 0; i < ANALOG_QTY; i++) {
    // Initialize the arrays
    analogReading[i] = 0;
    analogReadingPrev[i] = 0;
  }
  // start serial port at 9600 bps and wait for port to open:
  Serial.begin(9600);
  while (!Serial) {
    ; // wait for serial port to connect. Needed for native USB port only
  }
}

void loop() {
  for(int i = 0; i < ANALOG_QTY; i++) {
    // Read input
    analogReading[i] = analogRead(i);
    // Compare to previous one
    if (analogReading[i] != analogReadingPrev[i]) {
      // Send to computer if it changed
      analogReadingPrev[i] = analogReading[i];
      sendReading("analog", i, analogReading[i]);
    }
  }
  if (Serial.read() > 0) {
    // Send all current readings if we received something
    for(int i = 0; i < ANALOG_QTY; i++) {
      sendReading("analog", i, analogReading[i]);
    }
  }
  delay(10); // Enhance your calm
}

void sendReading(char type[], int i, int reading) {
  Serial.print(type);
  Serial.print(i);
  Serial.print(":");
  Serial.print(reading);
  Serial.print("\n");
}
```

This simply reads the analog pins (pins 0 through `ANALOG_QTY`), and prints the result on the serial port, if it changed from the previous reading. The data format is pretty simple: `analog<PIN_NUMBER>:<READING>`. I'll probably need something else than `analog` at one point, but that's all I have for now.

I tried this with a simple potentiometer while looking at the Arduino IDE serial window, and it seemed to work pretty well.

Now, how do we get this data into SignalK?

## Node.js program

If you've read the other article on this blog, you might guess that I'm a JavaScript developer, and you'd be correct. (Don't get me wrong, I know other languages, but I've used JS in my day job for most of my career.) So obviously, when I need to write a program, I mostly use node.js.

I used the `serialport` package from [npm](https://www.npmjs.com/package/serialport), and after a while, I came up with this:

```javascript

var config = {
  serialPort: "/dev/ttyUSB0",
  context: "vessels.urn:mrn:signalk:uuid:<UUID>",
  sensors: {
    analog0: {
      mapping: "electrical.dc.batteries.house.voltage",
      transform: function(input) {
        return input / 40.92;
      }
    },
    /* ... */
  }
};


var serialport = require("serialport");
var SerialPort = serialport.SerialPort;

console.error('Connecting to serial port');

var serialPort = new SerialPort(config.serialPort, {
  baudrate: 9600,
  parser: serialport.parsers.readline("\n")
}, false);

function onOpen() {
  console.error('Serial port connected');
  serialPort.write('1');
  serialPort.on('data', function(data) {
    var parsed = data.split(':');
    var sensor = parsed[0];
    if (config.sensors[sensor]) {
      var value = parseInt(parsed[1]);
      result = {
        "context": config.context,
        "updates": [
          {
            "source": {
              "device": config.serialPort,
              "timestamp": new Date(),
            },
            "values": [
              {
                "path": config.sensors[sensor].mapping,
                "value": config.sensors[sensor].transform(value)
              },
            ]
          },
        ]
      };
      console.log(JSON.stringify(result));
    }
  });
}

function onError(err) {
  console.error('Error connecting, retrying in 5 seconds...');
  setTimeout(function() {serialPort.open()}, 5000)
}

function onClose() {
  console.error('Connection closed, retrying in 5 seconds...');
  setTimeout(openSerial, 5000)
}

function openSerial() {
  serialPort.on("open", onOpen);
  serialPort.on('error', onError);
  serialPort.on('close', onClose);

  serialPort.open();
}

openSerial();
```

Most of it is housekeeping for the serial port (it grew significantly when I added proper error handling), but the two main things of note are the `config` object at the top, where I define a mapping from each sensor to a SignalK value, through a transform function, and the big object in the middle, which is creating a SignalK Delta message (see [here](http://signalk.org/developers/message_formats.html) for the spec) and printing it on the standard output.

I had an earlier version that connected to a TCP server, but then I discovered that there's a provider that executes programs and takes their output as input; it's a lot simpler that way.

It's pretty easy to test: put the proper serial port configuration for your Arduino, and start the program. You should see some JSON data scrolling by as the values read by the Arduino change.

## Putting it all together

The final step is to get my SignalK delta messages into the SignalK server. Here's the config file I created:

```json
{
  ...

  "pipedProviders": [{
    "id": "arduino-signalk",
    "pipeElements": [{
      "type": "providers/execute",
      "options": {
        "command": "node <PATH_TO_MY_NODEJS_SCRIPT>"
      }
    }, {
      "type": "providers/liner"
    }, {
      "type": "providers/from_json"
    }]
  }]
}
```

As you can see, it's pretty simple: I use the `execute` provider to run my program and gather its output, then the `liner` provider to split the messages neatly on line breaks (needed in case there's some buffering and the messages aren't batched exactly as I sent them), and the last one to parse my JSON messages.

At this point, you can start the SignalK server, and it should automatically start the node.js program, receiving data from your Arduino!

That's it for now! In the next part of the series, we'll look at making it more robust: udev rules for the Arduino, and running the SignalK server as a service.
