# IPNS-Link Specifications



### Main goals

1. Make any http-server addressable by an [IPNS](https://docs.ipfs.io/concepts/ipns/) name or key. This will be referred to as ***exposing an origin-server***.
2. Access the server using any local or public gateway.



### Participating nodes

1.  ***Origin*** : The http-server that is to be exposed. Each Origin is assigned a unique Libp2p-keypair. A hash of the corresponding public key becomes the IPNS name that the Origin-server would ultimately be addressed by. **Note** : By ***IPNS-name*** or ***IPNS-key***, henceforth, we shall refer to this hash.
2.  ***Listener*** : An IPFS node that listens for incoming connections over p2p-streams and forwards them to the Origin-server(s). The protocol name is **/x/ipns-link/`IPNS-key`**
3.  ***Publisher*** : An IPFS node whose sole job is to periodically publish a ***Manifest*** for each Origin, containing the [multiaddress](https://docs.libp2p.io/concepts/addressing/)es of the Listener and metadata about the Origin. The Manifest is published using [IPNS pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub) under the IPNS-key of the corresponding Origin. **Note** : The same IPFS node may perform as both Listener and Publisher. However, separating the nodes helps reduce bandwidth consumption.
4.  ***(Proxy) Gateway*** : An IPFS-aware http-proxy that, when given an IPNS-Key, connects the web-site visitors / end-users to the corresponding Origin via the Listener, after processing the Manifest published under that key. **Note** : This is distinct from an [IPFS-gateway](https://docs.ipfs.io/concepts/ipfs-gateway/), which can only serve static content from IPFS. The proxy Gateway can either be hosted locally by the end-user or publicly by a third-party.
5.  ***Browser*** : Any user-agent, such as the browser, cURL, HTTPie, Postman etc. that can make http(s) requests to a Gateway on behalf of the end-user.



### Manifest

##### Considerations

- The Manifest contains all the information a Gateway needs to discover and connect to the Origin, through the Listener. In addition, it contains some instructions for the Gateway, set by the owner of the Origin. 
- To ensure rapid dissemination, the Manifest is added to IPFS as **inline** and published using **IPNS pubsub**. Inlining encodes the entire Manifest inside the [CID](https://docs.ipfs.io/concepts/content-addressing/), so that no file needs to be imported separately after resolving the IPNS record. For efficient inlining, the Manifest needs to be as small in size as possible.
- It is desirable that an IPFS-gateway be able to redirect the Browser to a proxy Gateway, when provided with the IPNS-key of an exposed Origin. However, an IPFS-gateway can only deliver the Manifest file to the Browser. The Manifest, therefore, needs to contain an html redirect to a proxy Gateway.
- Because Gateways can always peek into the traffic between an end-user and the Origin, the Origin should have the liberty to choose only certain public Gateways as trusted. All the other Gateways would not be able to proxy for the Origin, but be able to redirect the Browser to one of the trusted Gateways.
- In case the Origin cannot trust any public Gateway, there should be a way to ensure that access is allowed through users' private Gateways only. Such a Gateway may be hosted by the user on localhost or on a personal cloud or VPS. Because private gateways may not have a public address, the Origin cannot provide any trusted Gateway URL in this case. However, the Origin might instead specify a certain webpage, containing instructions for the users, that the public Gateways shall redirect to.

##### Template

In view of the above, the Manifest is specified to be a small static site with nothing but a single `index.html`, henceforth referred to as **index**. The template for index is as follows.

```html
<meta http-equiv="refresh" content="0; url=https://gateway.tld/ipns/Key">

<title>Redirecting...</title>

If you are not redirected automatically, follow this 
<a href='https://official-gateway.tld/ipns/Key'>link</a>

<!--IPNS-Link--
ciphertext in multibase (base64, inline)
https://trusted-gateway1.tld
https://trusted-gateway2.tld
--IPNS-Link-->
```

The *meta* tag in the template above is meant for automatic redirect to a proxy Gateway from an IPFS-gateway. The *anchor* is meant for manual redirect to the official Gateway, in case the automatic redirect fails. A proxy Gateway itself, however, only processes the text inside the IPNS-Link comment block.

The comment block starts with a single line containing an **GPG-encrypted JSON** encoded in the [multibase](https://github.com/multiformats/multibase) format. To save space, it uses the base64 encoding. Following this line, are listed canonical URLs of all the Gateways trusted by the Origin, each in its own line. The JSON contains the Listener's multiaddresses as well as the metadata, as described [below](#json).

##### Encryption

Each Gateway owns a GPG key-pair and serves the corresponding public key at path `/pubkey`. The JSON is encrypted with the GPG public keys of the trusted Gateways. This makes sure only the trusted Gateways can read the JSON. Gateways that can't decrypt it, would [307 redirect](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307) the Browser to any one of the trusted Gateways, which may be chosen randomly.

If the Origin wants to be accessible through all the Gateways, then, instead of encrypting the JSON with the public keys, it is encrypted with an AES128 symmetric key with the passphrase `ipns-link`.

On the other hand, if the Origin wants to restrict access to the user's private Gateways only, the JSON is  encrypted symmetrically (AES128) with a shared secret. Because 3rd parties don't have access to the secret, public Gateways cannot decipher the all-important JSON. Such Gateways shall 307 redirect the Browser to any URL that may be provided in place of trusted Gateways in the Manifest. If no redirect target is there, a [401](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401) error code shall be issued.

##### JSON

The JSON contains the following key-value pairs by default. Values that must be non-null have been assigned imaginary values for illustration.

```json
{
  "cache": {
    "path": null,
    "CID": null
  },
  "on_fail": null,
  "stream": {
    "path": null,
    "keyID": null
  },
  "host": "hostname",
  "multiaddress": {
    "route": [      "/ip4/147.75.195.153/tcp/4001/p2p/QmW9m57aiBDHAkKj9nmFSEn7ZqrcF1fZS4bipsTCHburei/p2p-circuit",      "/ip4/147.75.195.153/udp/4001/quic/p2p/QmW9m57aiBDHAkKj9nmFSEn7ZqrcF1fZS4bipsTCHburei/p2p-circuit",
      "/ip4/13.235.23.238/tcp/4001"
    ],
    "ID": "12D3KooWN3z1MsVZoTYnVLKzu6rFz7xDaaW8iXuyn7AFXztZRYA3"
  }
}
```

Meaning of the key-value pairs are discussed below. In the following, the `value` of any `key` would be represented as `<key>`.

`cache` : The Gateway is meant to serve `GET` requests for all paths prefixed with `<cache.path>` from the immutable (static) directory at `/ipfs/<cache.CID>`. All these requests are therefore served from IPFS and need not reach the Origin, thus reducing latency and offloading Origin.

`on_fail` : On failure to connect (peer) with the Listener, the Gateway is to 307 redirect the browser to `<on_fail>`. The redirect destination might be a URL, an IPFS path `/ipfs/*` or an IPNS-key `/ipns/*`. If `null` or empty, the Gateway shall instead issue a [504](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/504) error code.

`stream` :  The Gateway is meant to serve `GET` requests for all paths prefixed with `<stream.path>` from the mutable (dynamic) directory at `/ipns/<stream.keyID>`. All these requests are therefore served from IPFS and need not reach the Origin, thus reducing latency and offloading Origin.

`host` : Instructs the Gateway to use `<host>` in the Host header while connecting to Origin.

`multiaddress` : Contains the complete [multiaddress](https://docs.libp2p.io/concepts/addressing/)es of the Listener.

**Note** : More key-value pairs may be added in future.

##### Publishing with IPNS pubsub

```bash
# Generate inline CID
ipfs add --inline --inline-limit=10000 -wQn index.html
# Publish with IPNSKey corresponding to Origin
ipfs name publish --key=IPNSKey -Q CID
```



### IPNS-Link = Exposer + Gateway

It is convenient to think of the entire scheme as composed of an **Exposer** and the **Gateway**. The Exposer consists of the Listener and the Publisher nodes. In order to expose the Origin, one simply has to run the Exposer daemon and (optionally) provide it with a list of trusted Gateways and values of `cache`, `on_fail` etc.



### Security

IPFS uses [transport-encryption](https://docs.ipfs.io/concepts/privacy-and-encryption/#encryption) , viz. data is secure when being sent from one IPFS node to another. This makes the `Gateway <--> Origin` connection secure. The `Browser <--> Gateway` connection is secured if either

- The end-user hosts a local Gateway.
- The end-user uses an HTTPS enabled public Gateway.

[Here](/trustless.md) is a draft proposal for an optional TLS/SSL passthrough mode that would make `Browser <--> Origin` demonstrably end-to-end encrypted.



### Bandwidth savings

If an IPFS node stays connected to the [WAN-DHT](https://docs.ipfs.io/concepts/dht/#dual-dht) for hours, it usually results in huge bandwidth consumption, even with [`Routing.Type=dhtclient`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype). 

To [save on bandwidth](https://www.digitalocean.com/blog/its-all-about-the-bandwidth-why-many-network-intensive-services-select-digitalocean-as-their-cloud), the Listener might go online with [`Routing.Type=none`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype). This should not affect its ability to use [Autorelay](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#autorelay) for NAT-traversal, as long as its bootstrap list contains enough nodes. If NAT traversal is not required, one might even remove the bootstrap list.

The Publisher publishes the Manifest using IPNS pubsub. Unlike the Listener, therefore, it has to remain connected to the DHT. However, because it has to publish only periodically, say every 15 minutes, the Publisher may be paused while idling. This significantly cuts down bandwidth consumption.

The "pause while idling" strategy might also be applied in case of the Gateways. The IPFS backend of the Gateway might be resumed when receiving an incoming http-request, and put to sleep once served.

**Note** : Pause and play may be achieved using SIGSTOP and SIGCONT signals respectively.

**TBD** : The p2p-connection between Gateway and the Listener may be bandwidth-limited. This would force exposed sites to serve large static content using the `cache` feature of IPNS-Link, and dynamic content using `swarm`, ensuring efficiency.



### Gateway

How a model Gateway might work is detailed [here](/gateway.md).



### Implementations

[Prototype Exposer](https://github.com/ipns-link/ipns-link)

[Prototype Gateway](https://github.com/ipns-link/ipns-link-gateway)



[![Contribute](https://img.shields.io/badge/Contribute%20to-IPNS--Link-brightgreen)](https://github.com/ipns-link/contribute) 

