# EnigmaIoT

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/logo/logo%20text%20under.svg?sanitize=true" alt="EnigmaIoT Logo" width="50%"/>

## Introduction

**EnigmaIoT** is an open source solution for wireless multi sensor systems. It has two main components, multiple **nodes** and one **gateway**.

A number of nodes with one or more sensors each one communicate in a **secure** way to a central gateway in a star network using EnigmaIoT protocol.

This protocol has been designed with security on mind. All sensor data is encrypted with a random key that changes periodically. Key is unique for each node and dinamically calculated, so user do not have to enter any key. Indeed, all encryption and key agreement is transparent to user.

I designed this because I was seaching for a way to have a relatively high number of nodes at home. I thought about using WiFi butit would overload my home router. So I looked for an alternative. I thought about LoRa or cheap 2.4GHz modules but I wanted the simplest solution in terms of hardware.

ESP8266 microcontroller implements a protocol known as ESP-NOW. It is a point to point protocol, based on special WiFi frames, that works in a connectionless way and every packet is short in time. Because of this it eases to have  the node feeded with batteries to have totally wireless sensors.

But use of encryption on ESP-NOW limits the number of nodes to only 6 nodes. So I thought that I could implement encryption on payload but I found many problems I should solve to grade this as "secure enough".

## Project requirements

During this project conception I decided that it should fulfil this list of requirements.

- Use the simplest hardware, based on ESP8266.
- Secure by design. Make use of a secure channel for data transmission.
- Automatic dynamic key agreement.
- Do not require connection to the Internet.
- Able to use deep sleep.
- Enough wireless range for a house
- Support for a high number of nodes

## Features

- Encrypted communication
- Dynamic key, shared between one node and gateway. Keys are independent for each node
- Number of nodes is only limited by memory on gateway (56 bytes per node)
- Key is never on air so it is not interceptable
- Key expiration and renewal is managed transparently
- Avoid repeatability attack having a new random initialization vector on every message
- Automatic and transparent node attachment
- Avoid rogue node, rogue gateway and man-in-the-middle attack. **TO-DO**
- Plugabble phisical layer communication. Right now only ESP-NOW protocol is developed but you can easily add more communication alternatives
- When using ESP-NOW only esp8266 is needed. No more electronics apart from sensor.
- Optional data message counter to detect lost or repeated messages.
- Designed as two libraries (one for gateway, one for node) for easier use.
- Selectable crypto algorhithm
- Gateway does not store keys only on RAM. They are lost on power cycle. This protects system against flash reading attack. All nodes attach automatically after gateway is switched on
- Downlink available. If deep sleep is used on sensor nodes, it is queued and sent just after node send a data message. **TO-DO**
- Optional sleep mode management. In this case key has to be stored temporally. Normally RTC memmory is the recommended place, and it is the one currently implemented, but SPIFFS or EEPROM would be possible. **Under test**

## Design

### System Design

System functions are divided in three layers: application, link and physical layer.

![Software Layers](https://github.com/gmag11/EnigmaIOT/raw/master/doc/system_layers.png)

- Application layer is not controlled by EnigmaIoT protocol but main program. User may choose whatever data format. A good option is to use CayenneLPP format but any other format or even raw data may be used. The only limit is the maximum packet length that, for ESP-NOW is around 200 bytes.
- Link layer is the one that add privacy and security. It manages connection between nodes and gateway in a transparent way. It does key agreement and node registration and checks the correctness of data messages. In case of any error it automatically start a new registration process. On this layer, data packets are encrypted using calculated symetric key.
- Physical layer currently uses connectionless ESP-NOW. But a hardware abstaction layer has been designed so it is possible to develop interfaces for any other layer 1 technology like LoRa or nRF24F01 radios.

### EnigmaIoT protocol

The named EnigmaIoT protocol is designed to use encrypted communication without the need to hardcode the key. It uses Diffie Hellman algorythm to calculate a shared key.

The process starts with node anouncing itself with a Client Hello message. It tells the gateway its intention to establish a new shared key. It sends Diffie Hellman public part to be used on gateway to calculate the key.

Gateway answers with Server Hello message that includes its public data for key calculation on node.

Once key is caliculated, node send an encrypted message as Key Exchange Finished message. A 32 bit CRC is calculated and it is used for decryption test.

If gateway validates CRC correctly it answers with a Cipher Finished message. It carries a CRC too.

In case of any error in the process gateway sends an Invalidate Key to reset to original status and forgets key.

When key is marked as valid node may start sending sensor data.

Optionally gateway can send data to node. As node may be sleeping between communications, downlink messages has to be sent just after uplink data. So, one downlink message is queued until node communicates. Node waits some milliseconds before sleep for downlink data.

Key is forced to change every period. Gateway decides the moment to invalidate each node key. If so, it sends an invalidate key as downlink, after next data message communication.

After that node may start new key agreement sending a new Client Hello message.

All nodes and gateway are identified by its MAC address. No name is assigned so no configuration is needed on node. Function assignment has to be done in a  higher level.

## State diagram for nodes and Gateway

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/StateDiagram-Sensor.svg?sanitize=true" alt="Sensor State Diagram" width="800"/>

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/StateDiagram-Gateway.svg?sanitize=true" alt="Gateway State Diagram" width="800"/>

## Message format specification

### Client Hello message

![Client Hello message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/ClientHello.png)

### Server Hello message

![Server Hello message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/ServerHello.png)

### Key Exchange Finished message

![Key Exchange Finished message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/KeyExchangeFinished.png)

### Cypher Finished message

![Cypher Finished message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/CypherFinished.png)

### Sensor Data message

![Sensor Data message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/SensorData.png)

### Sensor Command message (downlink)

![Sensor Command message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/SensorCommand-Downlink.png)

### Invalidate Key message

![Invalidate Key message format](https://github.com/gmag11/EnigmaIOT/raw/master/doc/InvalidateKey.png)

## Protocol procedures

### Normal node registration and sensor data exchange

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/NodeRegistration.svg?sanitize=true" alt="Normal node registration message sequence" width="400"/>

### Incomplete Registration

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/RegistrationIncomplete.svg?sanitize=true" alt="Incomplete Registration message sequence" width="400"/>

### Node Not Registered

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/NodeNotRegistered.svg?sanitize=true" alt="Node Not Registered message sequence" width="400"/>

### Key Expiration

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/KeyExpiration.svg?sanitize=true" alt="KeyExpiration message sequence" width="400"/>

### Node Registration Collision

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/NodeRegistrationCollision.svg?sanitize=true" alt="Node Registration Collision message sequence" width="600"/>

### Wrong Data Counter

<img src="https://github.com/gmag11/EnigmaIOT/raw/master/doc/WrongCounter.svg?sanitize=true" alt="Wrong Counter message sequence" width="400"/>

## Data format

Although it is not mandatory, use of [CayenneLPP format](https://mydevices.com/cayenne/docs/lora/#lora-cayenne-low-power-payload) is recommended for sensor data compactness.

You may use [CayenneLPP encoder library](https://github.com/sabas1080/CayenneLPP) on sensor node and [CayenneLPP decoder library](https://github.com/gmag11/CayenneLPPdec) for decoding on Gateway.

Example gateway code expands data message to JSON data, to be used easily as payload on a MQTT publish message to a broker. For JSON generation ArduinoJSON library is required.

In any case you can use your own format or even raw unencoded data. Take care of maximum message length that communications layer uses. For ESP-NOW it is about 200 bytes.