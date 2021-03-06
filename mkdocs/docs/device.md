# Device Client Package

The `ibmiotf.device` package includes the following:

- `ibmiotf.device.Client`
- `ibmiotf.device.ManagedClient`
- `ibmiotf.device.Command`
- `ibmiotf.device.DeviceInfo`
- `ibmiotf.device.DeviceFirmware`
- `ibmiotf.device.ParseConfigFile`
- `ibmiotf.device.ParseEnvVars`


## Client

```python
import ibmiotf.device

def myCommandCallback(cmd):
    print("Command received: %s" % cmd.data)


myConfig = ibmiotf.device.ParseConfigFile("device.yaml")

# Create an instance of the client
client = ibmiotf.device.Client(config=myConfig, logHandlers=None)

# Configure KeepAlive
client.setKeepAliveInterval(60)

# Register Command Handler
client.commandCallback = myCommandCallback

print("Connecting using keepalive interval of %s seconds" % (client.getKeepAliveInterval()) )
# Connect the device
client.connect()

# Create some data
myData={'name' : 'foo', 'cpu' : 60, 'mem' : 50}

# Publish an event
client.publishEvent(event="status", msgFormat="json", data=myData, qos=0, on_publish=None)

# Disconnect
client.disconnect()
```


## Configuration

The config parameter expects a dictionary in the following structure:

```python
{ "identity": {
        "orgId": "org1id",
        "typeId": "raspberry-pi-3"
        "deviceId": "00ef08ac05"
    }.
    "auth" {
        "token": "Ab$76s)asj8_s5"
    },
    "options": {
        "domain": "internetofthings.ibmcloud.com",
        "mqtt": {
            "port": 8883
            "transport": "tcp"
            "cleanSession": True
            "caFile": "/path/to/certificateAuthorityFile.pem"
        }
    }
}
```

- `identity.orgId` Your organization ID.
- `identity.typeId` The type of the device. Think of the device type is analagous to a model number.
- `identity.deviceId` A unique ID to identify a device. Think of the device id as analagous to a serial number.
- `auth.token` An authentication token to securely connect your device to Watson IoT Platform.
- `options.domain` Optional. A boolean value indicating which Watson IoT Platform domain to connect to (e.g. if you have a dedicated platform instance). Defaults to `internetofthings.ibmcloud.com`
- `options.mqtt.port` Optional. A integer value defining the MQTT port.
- `options.mqtt.transport` Optional. The transport to use for MQTT connectivity - `tcp` or `websockets`.
- `options.mqtt.cleanSession` Optional. A boolean value indicating whether to use MQTT clean session.  Defaults to `True`
- `options.mqtt.caFile` Optional. A String value indicating the path to a CA file (in pem format) to use in verifying the server certificate.  Defaults to `messaging.pem` inside this module


In most cases you will not manually build the `config` dictionary, we support two methods to generate device configuration:

### 1. From a Configuration File 

`ParseConfigFile()` allows one to easily pass in device configuration from environment variables.

```python
import ibmiotf.device

myConfig = ibmiotf.device.ParseConfigFile("device.yaml")
client = ibmiotf.device.Client(config=myConfig, logHandlers=None)
```

__Basic Configuration File__

```yaml
identity:
    orgId: org1id
    typeId: raspberry-pi
    deviceId: 00ef08ac05
auth:
    token: Ab$76s)asj8_s5
```

__Advanced Configuration File__ 

This file defines all optional configuration parameters.

```yaml
identity:
    orgId: org1id
    typeId: raspberry-pi
    deviceId: 00ef08ac05
auth:
    token: Ab$76s)asj8_s5
options:
    domain: internetofthings.ibmcloud.com
    mqtt:
        port: 8883
        transport: tcp
        cleanSession: true
        caFile: /path/to/certificateAuthorityFile.pem
```


### 2. From Environment Variables

`ParseEnvVars()` allows one to easily pass in device configuration from environment variables.

```python
import ibmiotf.device

myConfig = ibmiotf.device.ParseEnvVars()
client = ibmiotf.device.Client(config=myConfig, logHandlers=None)
```

- `WIOTP_ORG_ID`
- `WIOTP_TYPE_ID`
- `WIOTP_DEVICE_ID`
- `WIOTP_AUTH_TOKEN`
- `WIOTP_DOMAIN`
- `WIOTP_MQTT_PORT`
- `WIOTP_MQTT_TRANSPORT`
- `WIOTP_MQTT_CAFILE`
- `WIOTP_MQTT_CLEANSESSION`


## KeepAlive

`Client.setKeepAliveInterval()` allows the user to reconfigure the keepalive value for any subsequent MQTT connection made by the client.  
This does not affect the keep alive setting of any existing connection.  Traffic generated by keep alive is minimal, but
also billable as part of your data transfer to/from the Platform.


`Client.getKeepAliveInterval(numSeconds)` allows the user to retrieve the current setting for keepalive.  

Note: This is not necessarily the value used by the current active connection, as any changes to this value
are only applied when a new connection to the Platform is established


## Connectivity

`Client.connect()` & `Client.disconnect()` are used to manage the MQTT connection to IBM Watson IoT Platform that allows the device to 
handle commands and publish events.


## Publishing Events

Events are the mechanism by which devices publish data to the Watson IoT Platform. The device
controls the content of the event and assigns a name for each event that it sends.

When an event is received by Watson IoT Platform, the credentials of the received event identify
the sending device, which means that a device cannot impersonate another device.

Events can be published with any of the three quality of service (QoS) levels that are defined
by the MQTT protocol. By default, events are published with a QoS level of 0.

`Client.publishEvent()` takes up to 5 arguments:

- `event` Name of this event
- `msgFormat` Format of the data for this event
- `data` Data for this event
- `qos` MQTT quality of service level to use (`0`, `1`, or `2`)
- `on_publish` A function that will be called when receipt of the publication is confirmed.

__Callback and QoS__

The use of the optional `on_publish` function has different implications depending
on the level of qos used to publish the event:

- qos 0: the client has asynchronously begun to send the event
- qos 1 and 2: the client has confirmation of delivery from the platform



## Handling Commands

When the device client connects, it automatically subscribes to any command that is specified for
this device. To process specific commands, you need to register a command callback method.

```python
def myCommandCallback(cmd):
    print("Command received: %s" % cmd.data)

client.commandCallback = myCommandCallback
```

The messages are returned as an instance of the `Command` class with the following attributes:

- `command`: Identifies the command
- `format`: Format that the command was encoded in, for example `json`
- `data`: Data for the payload converted to a Python dict by an impleentation of `MessageCodec`
- `timestamp`: Date and time that the event was recieved (as `datetime.datetime` object)

If a command is recieved in an unknown format or if a device does not recognize the format, the device
library raises `ibmiotf.MissingMessageDecoderException`.


## Custom Message Format Support

By default, the client library support encoding and decoding events and commands as `json` messages.  To add support
for your own custom message formats you can create and register implementations of `MessageCodec`:

MessageCodecs work for both commands and events.  To Implement a MessageCodec you must support two static class methods:

### Encoding

The job of the `encode(data, timestamp)` method is to take `data` (any python object) and optionally a `timestamp` (a `datetime.datetime` object) and 
return a String representation of the message ready to be sent over MQTT.

### Decoding

The job of `decode(message)` is to decode an incoming MQTT message and return an instance of `ibmiotf.Message`

```python
import yaml
import imbiotf.device
import ibmiotf.Message
import ibmiotf.MessageCodec

class YamlCodec(ibmiotf.MessageCodec):
    
    @staticmethod
    def encode(data=None, timestamp=None):
        return yaml.dumps(data)
    
    @staticmethod
    def decode(message):
        try:
            data = yaml.loads(message.payload.decode("utf-8"))
        except ValueError as e:
            raise InvalidEventException("Unable to parse YAML.  payload=\"%s\" error=%s" % (message.payload, str(e)))
        
        timestamp = datetime.now(pytz.timezone('UTC'))
        
        return ibmiotf.Message(data, timestamp)

myConfig = ibmiotf.device.ParseConfigFile("device.yaml")
client = ibmiotf.device.Client(config=myConfig, logHandlers=None)
client.setMessageCodec("yaml", YamlCodec)
myData = { 'hello' : 'world', 'x' : 100}
# Publish the same event, in both json and yaml formats:
client.publishEvent("status", "json", myData)
client.publishEvent("status", "yaml", myData)
```

If you want to lookup which encoder is set for a specific message format call `client.getMessageEncoderModule(msgFormt)`:

If an event is sent in an unknown format or if a device does not recognize the format, the device
library raises `ibmiotf.MissingMessageEncoderException`.

## ManagedClient
