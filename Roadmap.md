# IPNS-Link Roadmap

## Implementations

Having Exposers and Gateways implemented in full for Go, JS, and other languages will yield significant improvements; they'll help increase stability, efficiency, portability and will allow the community to contribute to the project much more easily.

|Participant    |Language    |Status        |Repo
|:---            |:---        |:---        |:---
|Exposer        |Go            |Not started|none
|Gateway        |Go            |In-progress|https://github.com/ipns-link/go-gateway
|Exposer        |JS            |Not started|none
|Gateway        |JS            |In-progress|https://github.com/ipns-link/js-gateway

## Static Gateways

Static Gateways can help eliminate centralisation/federation from Public Gateways and improve many aspects of what IPNS-Link Gateways offer, they can also make the deployment and selfhosting of Public Gateways much easier. Additionally, they get around the MITM proxy vulnerability with TLS Passthrough, Static Gateways are verifiable and can be trusted based on their CID/IPNS address.

|Participant            |Language    |Status        |Repo
|:---                    |:---        |:---        |:---
|Static Public Gateway    |JS            |Not started|none
|Static Manifest Gateway|JS            |Not started|none
