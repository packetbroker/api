# Packet Broker API

Packet Broker is a packet-based traffic exchange for LoRaWAN networks.

This repository contains the API for interacting with the Packet Broker Router and Key Exchanges. The API is based on [gRPC](https://grpc.io/) for high performance and portability to any environment.

## Terminology

| Term | Meaning |
| --- | --- |
| Router | Routes message from Forwarders to Home Networks according to the Forwarder's Policy and Home Network's Filters |
| Key Exchange | Mechanism to transfer encryption Keys from Forwarders to Home Networks |
| Key | Encryption key. Can expire on a time and/or on a number of usages. Has a price per Key or per use. Key expiration is enforced by Key Exchange |
| Member | Authority of Home Networks and Forwarders. Manages its Home Networks with their `DevAddr` prefixes and its Forwarders. It must have a LoRa Alliance `NetID` |
| Forwarder | Network that forwards traffic from its gateways to and from Packet Broker. Has an ID within the Member scope |
| Home Network | Network where the device is registered and which manages the MAC state. Has a set of `DevAddr` prefixes |
| Policy | Routing policy of Forwarder and Home Network. Contains whether routing is enabled, Uplink Policy and Downlink Policy |
| Uplink Policy | Flags to indicate what uplink messages and metadata get forwarded and if downlink is allowed. Can be a combination of `NONE`, `MAC_DATA`, `APP_DATA`, `LOCALIZATION` (gateway location and fine timestamp if available), `SIGNAL_QUALITY` (gateway location, RSSI and SNR), `DOWNLINK_ALLOWED` |
| Downlink Policy | Flags to indicate what downlink messages can be forwarded. Can be a combination of `NONE`, `MAC_DATA`, `APP_DATA` |
| Filter | Filter uplink messages optionally by Member, Forwarder ID, `DevAddr` prefix, `FPort`, `FCnt`, only confirmed uplink, `FPorts` available. There can be multiple filters; one must pass to forward |

## Concept

### High level concept

- Forwarders generate Keys along with a key ID. Forwarders can use any rotation and key sharding strategy (per message, per time unit, per Home Network `NetID`, a combination of those, etc)
- Forwarders publish the Key and Key ID to a Key Exchange
- Forwarders choose a payload Key and a metadata Key (can be the same)
- Forwarders generate a random payload message key and metadata message key
   - The payload gets encrypted with the payload message key
   - The metadata gets encrypted with the metadata message key
   - The message keys get encrypted with the concerning Keys that were chosen
   - The encrypted keys are added to the message
- Forwarders generate the payload hash in the message
- Forwarders publish the encrypted message to Packet Broker Router. The message contains:
   - Encrypted payload (encrypted with the payload message key)
   - Encrypted payload message key (encrypted with the payload Key)
   - Payload Key ID the pointer to the Key Exchange
   - Encrypted metadata (encrypted with the metadata message key)
   - Encrypted metadata message key (encrypted with the metadata Key)
   - Metadata Key ID and the pointer to the Key Exchange
   - Payload hash (SHA256)
   - Payload length
   - LoRaWAN public fields: confirmed uplink yes/no, `FCnt`, `FPort`, `DevAddr`, `FOpts` yes/no
   - UL token
- Packet Broker Router routes the encrypted message with the Forwarder ID to the Home Network according to the Policy
- Home Network can buy the message, i.e. based on price(s) on Key Exchange(s) and payload hash (to check if the message has been received already)
- If the Key Exchange provides Key, the Home Network can decrypt the message itself. Otherwise, Home Network requests the Key Exchange to decrypt the message. The Home Network may optionally provide the `FNwkSIntKey` or `NwkSKey` to validate the MIC before recording the transaction
- Downlink works similar to stateless passive roaming
   - See Backend Interfaces 1.x `XmitDataReq`
   - Next to the Member's `NetID` (part of `XmitDataReq`), Home Networks need to copy the Forwarder ID along with the UL token

### Components

- Forwarder
   - Configures routing
      - Policies per Home Network
      - A default Policy (default is routing enabled with all uplink and downlink flags set)
   - Generates Keys
   - Publishes Keys to Key Exchanges
   - Encrypts payload and metadata
   - Publishes messages to its uplink topic on Packet Broker Routers
   - Subscribes to its downlink topic on Packet Broker Routers
- Packet Broker Router to route traffic from Forwarders to Home Networks and back. Acts as a switchboard
   - Forwarder facing
      - Topic based messaging service
      - Each forwarder has topics for uplink and downlink
   - Home Network facing
      - Topic based messaging service
      - Each Home Network has topics for uplink and downlink
   - Router
      - Uplink
         - Acts on messages published on a Forwarder topic
         - Determines the Home Network by the `NetID` from `DevAddr`. Drops the message if not found
         - Looks up the Uplink Policy of the concerning Forwarder and Home Network. Falls back to the default Policy. Drops the message if uplink routing is disabled
         - Applies the Uplink Policy (i.e. drop if `FPort > 0` and there's no `APP_DATA`, unset metadata)
         - Looks up the Home Network's Filters. If any, drop the message if any Filter doesn't pass
         - Publishes the message to the Home Network's topic
      - Downlink
         - Acts on messages published on a Home Network topic
         - Determines the Forwarder by the `NetID` and Forwarder ID. Drops the message if not found
         - Looks up the Downlink Policy of the concerning Forwarder and Home Network. Falls back to the default Policy. Drops the message if downlink routing is disabled
         - Applies the Downlink Policy
         - Publishes the message to the Forwarder's topic
- Home Network
   - Configures subscriptions
      - A set of Filters (default is a catch-all Filter)
   - Subscribes to its topic on Packet Broker Routers
   - Makes consideration to buy the message
   - Tries to obtain the Key from the Key Exchange
   - If the Key Exchange provides the Key, decrypts the message itself. Otherwise, request Key Exchange to decrypt
- Packet Broker Key Exchange
   - Sells the Key if it has a price. This allows Home Networks to decrypt messages offline. Forwarders are free to rotate Keys
   - Decrypts on request by the Home Network, respecting the Key usage limit, charging the Key usage price, optionally checks MIC if session key is provided

### Types of Key Exchanges

1. **Marketplace**
   Keeps track of messages from a Forwarder decrypted on request by a Home Network. Members have one billing relationship with the Marketplace for debit and credit invoices

2. **Balancer**
   Keeps track of messages from a Forwarder decrypted on request by a Home Network. Provides an overview of the balance. Members can offset the balance with a mutation that they both agree on, typically from an out-of-band transaction
