# IPNS-Link Specifications

![Projects](https://img.shields.io/badge/Projects-ipns--link%2C%20ipns--link--gateway-blue) ![PR](https://img.shields.io/badge/PRs-Accepted-brightgreen)

## Table of Contents  
[![tocgen](https://img.shields.io/badge/Generated%20using-tocgen-blue)](https://github.com/SomajitDey/tocgen)  
  - [IPNS-Link Specifications](#ipns-link-specifications)  
      - [Why IPNS-Link](#why-ipns-link)  
      - [Schema](#schema)  
          - [Participating nodes](#participating-nodes)  
          - [What Origin-Server does](#what-origin-server-does)  
          - [What IPNS-Link-Gateway does](#what-ipns-link-gateway-does)  
            - [X-Forwarded headers](#x-forwarded-headers)  
            - [Hiding client IP](#hiding-client-ip)  
            - [Cookies](#cookies)  
          - [What User-agent does](#what-user-agent-does)  
      - [IPNS-Link-Manifest](#ipns-link-manifest)  
          - [Template](#template)  
          - [Adding to IPFS](#adding-to-ipfs)  
          - [Publishing with IPNS with minimal bandwidth consumption](#publishing-with-ipns-with-minimal-bandwidth-consumption)  
      - [Security](#security)  
          - [Manifest encryption](#manifest-encryption)  
          - [Trusted Gateways](#trusted-gateways)  
      - [Intermittency](#intermittency)  
      - [Live streaming](#live-streaming)  
          - [References](#references)  
          - [Role of Origin-server](#role-of-origin-server)  
          - [Role of Gateway](#role-of-gateway)  
          - [Efficiency](#efficiency)  
      - [.ipns domains](#ipns-domains)  
      - [Benefits](#benefits)  
          - [Censorship resistance](#censorship-resistance)  
          - [Anonymity](#anonymity)  
          - [Dynamic IP address](#dynamic-ip-address)  
          - [Effective CDN](#effective-cdn)  
          - [Microhosting](#microhosting)  
          - [Lightweight mobile server](#lightweight-mobile-server)  
          - [Livestreams](#livestreams)  
          - [Free domain names](#free-domain-names)  
      - [Implementations](#implementations)  
      - [Contribute](#contribute)  
#####   

### Why IPNS-Link

[One can host a static website using IPFS+IPNS](https://medium.com/pinata/how-to-easily-host-a-website-on-ipfs-9d842b5d6a01). But what about a dynamic website or web app that needs to run server-side code! IPNS-Link aims to make it possible to expose http-servers using IPFS+IPNS.

### Schema

![IPNS-Link_schema](./IPNS-Link_schema.jpg)

##### Participating nodes

1.  ***Origin-server*** : An IPFS node on the same host that the http-server / web app runs on.
2. *(Public) **IPNS-Link-Gateway*** : IPFS-aware http-proxy between Origin-server and User-agent (defined below). Note that this is distinct from [IPFS-Gateways](https://docs.ipfs.io/concepts/ipfs-gateway/), which only serve static content from IPFS. ***Gateway***, henceforth, shall refer to IPNS-Link-Gateway, if not otherwise mentioned.
3. ***User-agent*** : Browser, cURL, HTTPie, Postman etc. that can make http(s) requests to the Gateway.

##### What Origin-Server does

1. Makes sure it is accessible from the public internet. Sets up [NAT-traversal](https://docs.libp2p.io/concepts/nat/), if needed.

2. Forwards incoming [libp2p-streams](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) with the `http` protocol to the local TCP port that the http-server runs on.

3. (Optional) Has a cache (directory) hosted on IPFS. This shall be referred to as ***IPFS-Cache*** henceforth. Hosts all the static contents (images, logos, header, footer, script, css etc.) either on [IPFS](https://docs.ipfs.io/how-to/address-ipfs-on-web/#protocol-upgrade) or inside the IPFS-Cache.

4. Publishes a small static website on IPNS, called ***IPNS-Link-Manifest*** (simply ***Manifest*** for short), that contains the following. (See the [corresponding section](#ipns-link-manifest) for more details).

   1. HTML redirect to an IPNS-Link-Gateway. This is so that when this website is accessed using an IPFS-Gateway, the User-agent is redirected to an IPNS-Link-Gateway.
   2. Publicly reachable multiaddress(s) of the Origin-Server. (Find rationale [here](#publishing-with-ipns-with-minimal-bandwidth-consumption)).
   3. Path prefix and [CID](https://docs.ipfs.io/concepts/content-addressing/) for the IPFS-Cache, if any.
   4. Path prefix and `KeyID` for HLS streams. (Details [here](#live-streaming)).
   5. An IPNS or IPFS path that the User-agent shall be redirected to, when the Origin-server is down. (More [here](#intermittency)). **Note**: If Origin-server is down for a long time (> 1h) this would be of no use, as the Manifest itself would be unavailable then.
   6. Further information, related to [security](#security).

   The Manifest would be published periodically, under the [PeerID](https://docs.libp2p.io/concepts/peer-id/) (Self key) of the Origin-server, using [IPNS Pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub).

5. For [HTTP Live Streams](#live-streaming), puts the segment files inside a directory and publishes the directory with IPNS Pubsub under a secondary key (described as `KeyID` above). 

6. Shows the site-owner its `PeerID` (*Self* key) that the owner can now distribute. If the owner owns a domain name, she may point to `/ipns/PeerID` as a [DNSLink](https://dnslink.io/) and create a CNAME record for her domain that points to her chosen IPNS-Link-Gateway.

##### What IPNS-Link-Gateway does

1. Receives an incoming request from a User-agent and filters it through a [rate-limiter](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429). 

2. Resolves the `PeerID` of the requested Origin-server from the request-path (resolution-type: path) or the `Host:` header (resolution-type: [subdomain](https://docs.ipfs.io/how-to/address-ipfs-on-web/#http-gateways) / [DNSLink](https://docs.ipfs.io/how-to/address-ipfs-on-web/#http-gateways)). [If resolving from path, redirects to appropriate subdomain in order to provide origin-isolation].

3. Accesses the Manifest published by the Origin-server from the IPFS network.

   > **TBD**: If no Manifest is found, and the request-path or `Host:` header contains a DNS name, the Gateway tries to resolve its [dnsaddr](https://github.com/multiformats/multiaddr/blob/master/protocols/DNSADDR.md).

   On failure, redirects to static content at `/ipns/PeerID/*` with a [`303`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303) response code.

4. Tries to connect to the Origin-server using the multiaddress(s) in the Manifest. If unable, redirects to the [`on fail` path mentioned in the Manifest](#ipns-link-manifest).

5. Forwards all HTTP requests from the User-agent to the Origin-Server as a [p2p http proxy](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#p2p-http-proxy) **except** for the following cases (listed in order of precedence):

   1. The request-method is `GET` and the request-path is a content-addressed IPFS path of the form: `/ipfs/*`. In this case, the User-agent is redirected to the corresponding path in the IPFS-world with a `303` code. **Note**: This happens regardless of any URL parameter/query string present.
   2. The request-path is an IPNS-Link of the form: `/ipns/*`. Here, there's a [`307`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307) redirect to an IPNS-Link-Gateway without any modification to the path, method or body. **However**, if the path is of the form `/ipns/KeyID/*`, where `KeyID` matches the [KeyID for HLS streams in the Manifest](#ipns-link-manifest), then the result of `ipfs cat <path>` is sent to the User-agent. See [here](#role-of-gateway) for more.
   3. The request-method is `GET` and the request-path matches the path-prefix corresponding to the IPFS-Cache. In this case, the path-prefix is replaced with the IPFS path of the cache and the User-agent is redirected to the resulting path with a `303` code.
   4. The request-method is `GET` and the request-path matches the path-prefix corresponding to the HLS stream. In this case, the path-prefix is replaced with the IPNS path of the stream directory and the User-agent is redirected to the resulting path with a `303` code.


Any redirect to an `/ipfs/*` path may be sent to an IPFS-Gateway (path or subdomain-type) for offloading.

###### X-Forwarded headers

When forwarding to the Origin-server, the Gateway must inject [`X-Forwarded-Host`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host), [`X-Forwarded-Port`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Port) and [`X-Forwarded-Proto`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Proto) headers.

###### Hiding client IP

The Gateway may or may not include any [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) header that can expose the originating IP address of the incoming requests to the Origin-server. Whether it masks the client IP should be mentioned clearly in the Gateway's homepage. [Anonymity can be verified dynamically, when maintaining a public list of public Gateways].

###### Cookies

The Gateway itself must not depend on cookies. This is so that the end-user can access the Gateway and therefrom sites that do not set cookies, even with the cookies blocked in her browser.

##### What User-agent does

IPNS-Link aspires to be compatible with all existing User-agents without requiring any special configuration or plugin. IPFS-aware User-agents, however, could be made to (launch and) use a local IPNS-Link-Gateway, instead of depending on a public one.

### IPNS-Link-Manifest

##### Template

The Manifest is actually a tiny static website with *only* a small `index.html` of the following form. Note how everything meant to be read by the IPNS-Link-Gateway(s) is in an inline base64 ciphertext embedded inside a single comment block titled IPNS-Link. The remaining lines inside that block may contain the [trusted gateway](#security) URLs (if any).

```html
<!--So that IPFS-Gateway redirects to a random IPNS-Link-Gateway-->
<meta http-equiv="refresh" content="0; url=https://gateway.tld/ipns/PeerID">

<title>Redirecting...</title>

<!--The official IPNS-Link-Gateway in case the above random one is down-->
If you are not redirected automatically, follow this 
<a href='https://gateway-official.tld/ipns/PeerID'>link</a>

<!--IPNS-Link--
base64 ciphertext string (inline)
https://trusted.gateway1.tld
https://trusted.gateway2.tld
--IPNS-Link-->

```

The ciphertext encrypts the following JSON:

```bash
{
  "cache": {
    "path": "/cache/path/prefix",
    "CID": "CIDHash"
  },
  "multiaddress": {
    "ID": "PeerID"
    "route": [
        "/ip4/1.2.3.4/tcp/4001/p2p/RelayID/p2p-circuit",
        "/ip4/1.2.3.4/udp/4001/quic"
    ]
  },
  "on_fail": "/ipns/AnotherOriginServerID or /ipfs/WillBeBackPage or https(s) URL",
  "stream": {
    "path": "/stream/directory/path",
    "keyID": "KeyIDHash"
  },
  "host": "hostname.tld"
}
```

The encryption is either asymmetric with the ed25519 GPG public key of the trusted gateway (see [Security](#security) for more), or symmetric (AES128) with the GPG passphrase: `ipns-link`. The purpose of the symmetric encryption is not security but compression and simplicity. The base64 ciphertext is generated by transforming the binary output of `gpg` to `ipfs multibase encode -b base64` - and not by making `gpg` output armored ascii. This is simply to reduce the overall size of the Manifest.

##### Adding to IPFS

The `index.html` is added to IPFS using

```bash
ipfs add --inline --inline-limit=600000 --wrap-with-directory -Q -n index.html | ipfs cid format -v1 -b base64
```

Thanks to the inlining, the Manifest can always be extracted from the CID that is output from the above command. The `index.html`, therefore, need not be provided by a local or remote pinning service constantly connected to the [WAN-DHT](https://docs.ipfs.io/concepts/dht/#dual-dht) and costing bandwidth.

##### Publishing with IPNS with minimal bandwidth consumption

An IPFS node usually stores its multiaddresses as *Peer records* in the [DHT](https://docs.ipfs.io/concepts/dht/#distributed-hash-tables-dhts). Staying connected to the [WAN-DHT](https://docs.ipfs.io/concepts/dht/#dual-dht) for hours, however, results in huge bandwidth consumption, even with [`Routing.Type=dhtclient`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype).

To [save on bandwidth](https://www.digitalocean.com/blog/its-all-about-the-bandwidth-why-many-network-intensive-services-select-digitalocean-as-their-cloud), Origin-server goes online with [`Routing.Type=none`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype). Note that this does not affect its ability to use [Autorelay](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#autorelay) for NAT-traversal, as long as enough bootstrap nodes are there. If NAT traversal is not required, however, one can safely do away with the bootstrap nodes.

Because it is disconnected from the WAN-DHT, the Origin-server must delegate the actual publishing of its IPNS records to an auxilliary IPFS node, henceforth called the ***Publisher***, that runs on the same host (or LAN) and mostly sleeps when not publishing, thus saving on bandwidth. When the Origin-Server prepares an IPNS record for its Manifest, it wakes the Publisher up, and passes the IPNS-records through a pipe, Unix-socket, tmpfile or LAN. The Publisher connects to the WAN-DHT, does an `ipfs dht put`, and goes to sleep again. Pausing and resuming the Publisher may be done with `SIGSTOP` and `SIGCONT` respectively. The Publisher must have IPNS Pubsub enabled.

When NAT traversal is required, the Publisher, by virtue of its connection to the WAN-DHT, might be used to find p2p-circuit relays faster. Once the Publisher has obtained a public multiaddress, the Origin-server can simply extract the multiaddress of the relay node from the multiaddress of the Publisher and puts the relay in its [peering subsystem](https://docs.ipfs.io/reference/cli/#ipfs-swarm-peering).

### Security

[IPFS uses transport-encryption, viz. data is secure when being sent from one IPFS node to another](https://docs.ipfs.io/concepts/privacy-and-encryption/#encryption). Therefore, `Gateway <--> Origin-server` connection is secure. In view of this, when using a public Gateway with `https`, the user ought to be secure. But this applies only if the Gateway, where TLS terminates, can be trusted. The issue of trust may be addressed in the following ways:

- The user hosts her own Gateway, on [localhost](http://localtest.me) (free-of-cost) or with her trusted cloud provider (costs PaaS, wildcard-subdomains and SSL-certs).
- By encrypting the Origin-server's `multiaddresses` with the public keys of a (few) trusted Gateway(s) and linking the trusted Gateway(s) in the Manifest's [index.html](#ipns-link-manifest). Details follow.

##### Manifest encryption

The Gateway node serves its ed25519 GPG public key at `http(s)://gateway.tld/pubkey`. With this premise, Manifest encryption works as follows:

1. Origin-server chooses a few Gateways that it trusts and retrieves their pubkeys. While creating the [Manifest](#ipns-link-manifest), it encrypts the JSON with these pubkeys to prepare the [`ciphertext`](#ipns-link-manifest). The array of Gateways (URLs) that can decipher the ciphertext is then put below the ciphertext in the ipns-link comment block within the index.html file.

2. An IPNS-Link-Gateway tries to read the Origin-server's manifest. If unable, it redirects the User-agent to one of the Gateways listed below the ciphertext.

4. The trusted Gateway decrypts the Manifest and [proceeds](#what-ipns-link-gateway-does).

##### Trusted Gateways

- The Gateway run by the end-user at `localhost`.
- The site owner (e.g. a company / bank) might run one or a few IPNS-Link-Gateway(s) for all of its Origin-servers. For added protection, in this case, all the Origin-servers and Gateways might share a swarm-key and be part of a [private network](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#private-networks).
- A trusted CDN provider like [Cloudflare](https://blog.cloudflare.com/distributed-web-gateway/) can have Gateways distributed across the globe.
- Gateways operated by [Protocol Labs](https://protocol.ai/).

### Intermittency

When an Origin-server goes offline, the Gateway redirects to the link provided against the `on_fail` key in [Manifest](#template). This link might take the User-agent to a backup Origin-server or a static website that keeps the visitor engaged or informed - e.g. the page might ensure the visitor that the site would be up shortly etc.

### Live streaming

##### References

- [Intro: HTTP Live Stream](https://developer.apple.com/streaming/)
- [HLS is supported on all major browsers, atleast with JavaScript](https://developer.mozilla.org/en-US/docs/Web/Guide/Audio_and_video_delivery/Live_streaming_web_audio_and_video)

##### Role of Origin-server

1. Puts all the segment files inside a directory. Publishes the directory over IPNS Pubsub everytime a new segment file is generated. The publishing key is something other than the *Self* key, hence its `KeyID` is different from the Origin-server's `PeerID`.
2. Puts the `KeyID` and the stream directory path in its [Manifest](#template).
4. Responds to requests for the playlist file that are forwarded by the Gateway.

##### Role of Gateway

1. When forwarding the request for the playlist to Origin-server, connects to it. This makes the Origin-server a direct peer of the Gateway.
2. When receiving forward requests for paths prefixed with the stream/directory/path, replace the prefix with `/ipns/stream_keyID/` and serve the IPNS-path as usual - viz. from cached IPLD blocks if possible, retrieving from Origin-server otherwise.

##### Efficiency

A stream segment is sent to the Gateway from the Origin-server only once. Thereafter, all requests for that segment are served by the Gateway directly from its cache.

### .ipns domains

See the corresponding [repo](https://github.com/ipns-link/dot-ipns-registry) for details. For validation purposes, there should be a `host` field in the [Manifest](#ipns-link-manifest).

### Benefits

##### Censorship resistance

A website that is blocked in a country or region may be accessed using *any* Gateway, including the one at localhost. If a list of public Gateways is maintained, a working link can be generated trivially as `https://gateway.tld/ipns/blocked.site`.

##### Anonymity

Accessing websites through [certain](#hiding-client-IP) public Gateways hides the user's IP address from the websites visited. Compare [Tor](https://www.torproject.org/) and [VPN](https://en.wikipedia.org/wiki/Virtual_private_network). As an aside, it may be noted that [IPFS is also experimenting with onion routing](https://dweb-primer.ipfs.io/avenues-for-access/tor-transport).

##### Dynamic IP address

Whenever the Origin-server detects a change in its public IP address, it simply publishes a new [Manifest](#ipns-link-manifest).

##### Effective CDN

Every `GET` request is not forwarded to the Origin-server; some are served from the [IPFS](#what-origin-server-does).

##### Microhosting

Anyone can host a server on a Raspberry Pi or an old PC and expose with IPNS-Link, readily, free of cost. Visitors can reach the site as long as they know the `PeerID`, making domain purchase optional. With built-in NAT-traversal, there's no need to pay for a public IP address either. When served through [trusted gateways](#trusted-gateways) only, using [Manifest encryption](#manifest-encryption), the server becomes essentially TLS protected (https) without the hassle of obtaining and managing SSL certificates.

##### Lightweight mobile server

Mobile devices remain on almost all the time. With [IPFS on mobile](https://github.com/ipfs-shipyard/gomobile-ipfs), or an IPNS-Link-desktop-app running on [Termux](https://termux.com/), a cell-phone or a tablet might act as an Origin-server. If most of the site content is static, then only a few requests need ever reach the Origin-server, allowing it to be lightweight. Because the Publisher node mostly sleeps, and the Origin-server remains disconnected from the WAN-DHT, there's not much going on at any given time, thus saving on battery. Also, [whenever the device goes offline from time to time](#intermittency), the site visitors maybe offered something instead of nothing.

##### Livestreams

With multiple Gateways, each visitor would choose a Gateway nearest to them. Since each Gateway serves the stream content from its IPLD cache, this would ensure a faster streaming experience. Also the Origin-server is significantly [offloaded](#efficiency).

##### Free domain names

The [proposed](https://github.com/ipns-link/dot-ipns-registry) `.ipns` domain name system is a free alternative to DNSLink, aimed to replace the long IPFS / IPNS paths with human-friendly names. As long as a Gateway can resolve such names, a website can be easily accessed with https://chosen-name.ipns.gateway.tld or https://any-gateway.tld/ipns/chosen.name.ipns

### Implementations

Coming up ...

### Contribute

Lots of things to be done. You are more than welcome to become an active part of [IPNS-Link](https://github.com/ipns-link) and help out anyway you can. [Here](https://github.com/ipns-link/contribute) are some suggestions.
