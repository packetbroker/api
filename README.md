# Packet Broker API

Packet Broker is a packet-based traffic exchange for LoRaWAN networks.

This repository contains the API for interacting with the Packet Broker Router and Key Exchanges. The API is based on [gRPC](https://grpc.io/) for high performance and portability to any environment.

## Terminology

| Term | Meaning |
| --- | --- |
| Router | Routes message from Forwarders to Home Networks according to the Forwarder's Policy and Home Network's Filters |
| Key Exchange | Mechanism to transfer encryption Keys from Forwarders to Home Networks |
| KEK | Key encryption key. Can expire on a time and/or on a number of usages. Has a price per key or per use. Key expiration is enforced by Key Exchange |
| DEK | Data encryption key. These are sent encrypted with the key encryption key |
| Member | Authority of Home Networks and Forwarders. Manages its Home Networks with their `DevAddr` prefixes and its Forwarders. It must have a LoRa Alliance `NetID` |
| Forwarder | Network that forwards traffic from its gateways to and from Packet Broker. Has an ID within the Member scope |
| Home Network | Network where the device is registered and which manages the MAC state. Has a set of `DevAddr` prefixes |
| Routing Policy | Routing policy of Forwarder and Home Network. Contains whether routing is enabled, Uplink Policy and Downlink Policy |
| Uplink Routing Policy | Flags to indicate what uplink messages and metadata get forwarded and if downlink is allowed. Can be a combination of MAC data, application data, signal quality (gateway antenna RSSI and SNR), localization (gateway antenna location, RSSI, SNR and fine timestamp if available), and whether downlink is allowed |
| Downlink Routing Policy | Flags to indicate what downlink messages can be forwarded. Can be a combination of MAC data and application data |
| Routing Filter | Filter uplink messages optionally by Member, Forwarder ID, confirmed yes/no, `DevAddr` prefix, `FOpts` yes/no, `FPort` ranges and whether or not gateway metadata is present. There can be multiple filters; any filter that passes has the message forwarded to the Home Network |

## Concept

### High level concept

On uplink:

- Forwarders publish an uplink message with encrypted fields to the Home Network via Packet Broker Router
- Forwarders generate one or multiple key encryption keys (KEKs) with a KEK label. Forwarders can use any rotation and key sharding strategy (per message, per time unit, per Home Network `NetID`, a combination of those, etc)
- Forwarders publish the KEKs to Key Exchanges
- Forwarders generate data encryption keys (DEKs) for `PHYPayload`, signal quality and localization metadata. The DEKs may be the same or may be different
- Forwarders encrypt the uplink message
   - The `PHYPayload` gets encrypted with the DEK for `PHYPayload`
   - The signal quality gets encrypted with the DEK for signal quality
   - The localization gets encrypted with the DEK for localization
   - Forwarders choose which KEKs to use to encrypt the DEKs
   - The encrypted DEKs are added to the message, referring to the KEK that encrypted the DEK
   - All encryption uses AES-256-GCM
- Forwarders generate the SHA-256 payload hash in the message
- Forwarders publish the encrypted message to Packet Broker Router. The uplink message contains:
   - Pointers to KEKs used for encrypting DEKs in the message. The pointers consist of the Key Exchange address and the KEK label
   - Teaser of the `PHYPayload` to drive the decision to decrypt an encrypted DEK, containing:
     - SHA-256 hash of the payload
     - LoRaWAN public fields: confirmed uplink yes/no, `DevAddr`, `FOpts` yes/no, `FCnt`, `FPort` and `FRMPayload` length
   - Encrypted `PHYPayload` and the DEK encrypted with one or multiple KEKs
   - Teaser of the gateway metadata to drive the decision to decrypt an encrypted DEK, containing:
     - Whether the gateway is terrestrial or a satellite
     - Indication whether localization data contains a fine timestamp
   - Encrypted signal quality and the DEK encrypted with one or multiple KEKs
   - Encrypted localization and the DEK encrypted with one or multiple KEKs
   - Forwarder and gateway uplink tokens
   - Region of the gateway as defined in LoRaWAN Regional Parameters
- Packet Broker Router routes the encrypted message with the Forwarder ID to the Home Network according to the Routing Policy and Routing Filter
- Home Network can buy the message (i.e. decrypt a DEK) based the teasers, based on price(s) on Key Exchange(s) and payload hash (to check if the message has been received already via other means)
- If the Key Exchange provides the KEK, the Home Network can decrypt the DEK itself. Otherwise, the Home Network requests the Key Exchange to decrypt the DEK
  - Alternatively, the Home Network may optionally have the `PHYPayload` decrypted and pass the `FNwkSIntKey` or `NwkSKey` to validate the MIC before recording the transaction, if the Key Exchange supports this

On downlink:

- The Home Network publish a downlink message to a Forwarder via Packet Broker Router
- The downlink message contains:
  - `PHYPayload`
  - RX1 and RX2 frequency and data rate index (in gateway region from the uplink message)
  - RX1 delay
  - Class (A, B or C)
  - Forwarder and gateway uplink tokens copied from the uplink message

Downlink is very similar to Stateless Passive Roaming as defined in LoRaWAN Backend Interfaces. The API is fully compatible with the `XmitDataReq` message.

### Components

- Forwarder
   - Configures routing
      - Routing Policies per Home Network
      - A default Routing Policy (default is routing enabled with all uplink and downlink flags set)
   - Generates KEKs and DEKs
   - Publishes KEKs to Key Exchanges
   - Encrypts DEKs with one or multiple KEKs
   - Encrypts `PHYPayload`, signal quality and localization metadata with DEKs
   - Publishes messages to Packet Broker Router
   - Subscribes for downlink on Packet Broker Router
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
         - Looks up the Uplink Routing Policy of the concerning Forwarder and Home Network. Falls back to the default Routing Uplink Policy. Drops the message if uplink routing is disabled
         - Applies the Uplink Routing Policy
         - Looks up the Home Network's Routing Filters. If any, drop the message if any Filter doesn't pass
         - Publishes the message to the Home Network's topic
      - Downlink
         - Acts on messages published on a Home Network topic
         - Determines the Forwarder by the `NetID` and Forwarder ID. Drops the message if not found
         - Looks up the Downlink Policy of the concerning Forwarder and Home Network. Falls back to the default Routing Policy. Drops the message if downlink routing is disabled
         - Applies the Downlink Routing Policy
         - Publishes the message to the Forwarder's topic
- Home Network
   - Publishes downlink messages to Packet Broker Router
   - Subscribes to uplink on Packet Broker Router;
      - A set of Routing Filters (if not specified, a catch-all Routing Filter applies)
   - Makes consideration to buy `PHYPayload` or metadata
   - On a positive decision, has the encrypted DEK decrypted
   - If the Key Exchange provides the KEK, decrypts the DEK itself. Otherwise, request Key Exchange to decrypt the encrypted DEK
- Packet Broker Key Exchange
   - Sells the KEK if it has a price. This allows Home Networks to decrypt messages offline. Forwarders are free to rotate KEKs and use multiple KEKs at any time
   - Decrypts on request by the Home Network, respecting the KEK usage limit, charging the KEK usage price, optionally checks MIC if session key is provided

### Types of Key Exchanges

All Key Exchanges keep track of the usage of KEKs to decrypt DEKs on request by a Home Network. Forwarders and Home Networks can be the same Member if they play both roles.

1. **Marketplace**
   Members have one billing relationship with the Marketplace for debit and credit invoices

2. **Balancer**
   Members can offset the balance with a mutation that they both agree on, typically from an out-of-band monetary transaction

## License

The API is distributed under [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). See `LICENSE` for more information.
