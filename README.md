# IPNS-Link Specifications

![Projects](https://img.shields.io/badge/Projects-ipns--link%2C%20ipns--link--gateway-blue) ![PR](https://img.shields.io/badge/PRs-Accepted-brightgreen)

## Table of Contents  
[![tocgen](https://img.shields.io/badge/Generated%20using-tocgen-blue)](https://github.com/SomajitDey/tocgen)  
  - [IPNS-Link Specifications](#ipns-link-specifications)  
      - [Why IPNS-Link](#why-ipns-link)  
      - [Basic principles](#basic-principles)  
          - [Schema](#schema)  
          - [Participating nodes](#participating-nodes)  
          - [What source node does](#what-source-node-does)  
          - [What gateway does](#what-gateway-does)  
      - [Where IPNS comes in](#where-ipns-comes-in)  
          - [Publishing multiaddress with minimal bandwidth consumption](#publishing-multiaddress-with-minimal-bandwidth-consumption)  
          - [Redirect to IPNS-Link-gateway](#redirect-to-ipns-link-gateway)  
      - [Template for the IPNS post](#template-for-the-ipns-post)  
          - [Notes](#notes)  
      - [IPNS-Link-gateway specs](#ipns-link-gateway-specs)  
          - [Path gateway](#path-gateway)  
      - [Benefits](#benefits)  
          - [Security and Trustlessness](#security-and-trustlessness)  
          - [Uncensored hosting](#uncensored-hosting)  
          - [Anonymity](#anonymity)  
          - [No need to pay for DDNS](#no-need-to-pay-for-ddns)  
          - [Low-cost hobby hosting](#low-cost-hobby-hosting)  
      - [Implementations](#implementations)  
      - [Contribute](#contribute)  
#####   

### Why IPNS-Link

[One can host a static website using IPFS+IPNS](https://medium.com/pinata/how-to-easily-host-a-website-on-ipfs-9d842b5d6a01). But what about a dynamic website or web app that needs to run server-side code! IPNS-Link aims to make it possible to expose http-servers using IPFS+IPNS.

### Basic principles

##### Schema

![IPNS-Link_schema](./IPNS-Link_schema.jpg)

##### Participating nodes

1. Source node (origin server) where the http-server (web app) runs.
2. (Public) gateway to access the http-server (from browser).

##### What source node does

1. Makes sure it is accessible from the public internet. Sets up [NAT-traversal](https://docs.libp2p.io/concepts/nat/), if needed.
2. Publishes its public [multiaddress](https://docs.libp2p.io/concepts/addressing/).
3. [Forwards incoming libp2p-streams with the `http` protocol to the local http-server.](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p)

##### What gateway does

1. Access the public multiaddress of the source node and connect.
2. Connect to the local http-server at the source node, as a [p2p http proxy](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#p2p-http-proxy).

### Where IPNS comes in

##### Publishing multiaddress with minimal bandwidth consumption

An IPFS node stores its multiaddresses as Peer records in the [DHT](https://docs.ipfs.io/concepts/dht/#distributed-hash-tables-dhts). Staying connected to the [WAN-DHT](https://docs.ipfs.io/concepts/dht/#dual-dht) for hours, however, results in huge bandwidth consumption, even with [`Routing.Type=dhtclient`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype).

The source node solves this in the following way. It goes online with [`Routing.Type=none`](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype). Note that this does not affect its ability to use [Autorelay](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#autorelay) for NAT-traversal, as long as enough bootstrap nodes are there. If NAT traversal is not required, however, one can do away with the bootstrap nodes.

Because it is disconnected from the WAN-DHT, the source node employs a secondary, ephemeral node to publish its DHT records to the public network periodically. This secondary node is ephemeral, in that it stays connected to the WAN-DHT only for a short while. Remaining online only for a few seconds for a few times every hour does not cost much bandwidth. 

An IPFS node, however, can put to DHT only the IPNS records of another node, not its [Provider or Peer records](https://docs.ipfs.io/concepts/dht/). The source node, therefore, puts its multiaddresses inside its [IPNS record](#template-for-the-ipns-post), `/ipns/PeerID`, and passes the record to the secondary node.

##### Redirect to IPNS-Link-gateway

The [gateway](#ipns-link-gateway-specs) that connects to the http-server at the source node is different from an [IPFS gateway](https://github.com/ipfs/go-ipfs/blob/master/docs/gateway.md) in the following ways. 

1. It needs to access the multiaddresses of the source node from IPNS instead of DHT.
2. It needs to allow `/p2p` paths.

To distinguish this gateway from an IPFS gateway, let's call it an [***IPNS-Link-gateway***](#ipns-link-gateway-specs).

Now consider this. What if someone tries the path `/ipns/PeerIDofSourceNode` using one of the public [IPFS gateways](https://ipfs.github.io/public-gateway-checker/), in a browser? Won't it be great if the visitor is redirected to an IPNS-Link-gateway that can actually connect to the live web app running at the source node? To achieve this, the source node also puts an html redirect inside its [IPNS record](#template-for-the-ipns-post).

### Template for the IPNS post

The post is actually a tiny static website with an `index.html` of the form:

```html
<meta http-equiv="refresh" content="0; url=https://example.com/ipns/PeerIDofSourceNode">
<title>Redirecting...</title>
If you are not redirected automatically, follow this <a href='https://example.com/ipns/PeerIDofSourceNode'>link</a>

<!--ipns-link--
/ip4/127.0.0.1/tcp/4001/p2p/PeerIDofRelay/p2p-circuit/p2p/PeerIDofSourceNode
--ipns-link-->

<!--
Other texts, if any
-->
```

Replace `example.com` with an IPNS-Link-gateway of your choice. Replace the multiaddress `/ip4/...` with the fully qualified public multiaddress(es) of the source node.

##### Notes

Add the post to IPFS using

```bash
ipfs add --inline --inline-limit=1000 --wrap-with-directory index.html
```

Thanks to the inlining, the IPNS record pointing to this tiny website actually contains the whole website. Because the IPNS record is cached by DHT nodes, IPFS gateways and [IPNS Pubsub peers](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub), the website doesn't need to be provided by the source node itself, nor do we need to use a remote pinning service like [Pinata](https://www.pinata.cloud/).

### IPNS-Link-gateway specs

1. Determine the source node's identifier ([PeerID](https://docs.libp2p.io/concepts/peer-id/#what-is-a-peerid) or any other [IPNS name](https://docs.ipfs.io/concepts/ipns/)) from the user request (see "Resolution style" below). Redirect, if necessary.
2. Access the [IPNS post](#template-for-the-ipns-post)  by the source node, and retrieve the multiaddresses by parsing it.
3. Connect to the source node using the retrieved multiaddresses.
4. Support all HTTP methods while forwarding incoming http-requests to the web app at the source node.
5. No [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) header that can expose the originating IP address of the incoming requests.
6. Might contain [`X-Forwarded-Host`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host), [`X-Forwarded-Port`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Port) and [`X-Forwarded-Proto`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Proto) headers.
7. Resolution style: Path, Subdomain and DNSLink. See [this](https://docs.ipfs.io/concepts/ipfs-gateway/#gateway-types) to familiarize yourself with these types.

##### Path gateway

Request types: 

1. `http(s)://gateway.tld/ipns/IPNSname/*` or  
2. `http(s)://gateway.tld/p2p/PeerID/http/*`

Multiple `IPNSname`s can point to the same `PeerID`. Therefore, all paths of type 1 are redirected to the path of type 2.

Path gateways are easier to deploy and maintain but provide no origin isolation. This causes problems on mainly two fronts, cookies and root-relative-URLs. We propose to solve these, albeit partially, with some header magic as follows.

1. **Root-relative-URL:** When you are at path `https://gateway.tld/p2p/PeerID/http/document` and you click a link for `/image`, the request path becomes `https://gateway.tld/image`. The gateway, therefore, needs some means to extract the `PeerID` which is now missing from the path. Note that it can use the [`Referer`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) header for this purpose. [To make this work always, the gateway may inject a `strict-origin-when-cross-origin` [`Referrer-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy) in the response message from the source node]. After extracting the `PeerID` from the `Referer` header, the gateway makes a final redirection to `/p2p/PeerID/http/image`.

2. **Cookies:** If source nodes are allowed to [set cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie) with [path attributes](https://datatracker.ietf.org/doc/html/rfc6265#section-5.1.4), cookies set by one source node might be sent to another. To mitigate this, the gateway may modify the cookie-paths by prefixing them with `/p2p/PeerID/http`. So, the header 

   ```html
   Set-Cookie: Name=Value; Path=/path
   ```

   from the source node with peer ID = `PeerID` becomes

   ```html
   Set-Cookie: Name=Value; Path=/p2p/PeerID/http/path
   ```

   This works only because the gateway always redirects to paths of type: `/p2p/PeerID/http/*` (see above). For added safety, the gateway might remove any `Domain` attribute in the [`Set-Cookie`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie) header.

### Benefits

##### Security and Trustlessness

[IPFS uses transport-encryption, viz. data is secure when being sent from one IPFS node to another](https://docs.ipfs.io/concepts/privacy-and-encryption/#encryption). So gateway <--> source node connection is secure. If you are using a public IPNS-Link gateway with `https`, you ought to be secure, but only as long as you trust the public gateway provider. If you don't want to trust the public gateways you can always [host](https://github.com/ipns-link/ipns-link-gateway) your own, for free, locally or on cloud.

##### Uncensored hosting

To illustrate, imagine the Sci-Hub server exposed using IPNS-Link. In countries where Sci-Hub is blocked, one can simply access it through *any* public IPNS-Link-gateway.

##### Anonymity

Accessing websites through public IPNS-Link-gateways hides your IP address from the websites visited. Compare Tor and VPN. As an aside, you may note that [IPFS is also experimenting with onion routing](https://dweb-primer.ipfs.io/avenues-for-access/tor-transport).

##### No need to pay for DDNS

Traditionally, if your server only had a dynamic public IP address, you would be forced to buy a DDNS service. With IPNS-Link, you can simply point your domain to `{your IPNS identifier}.{a public IPNS-Link-gateway URL}`.

##### Low-cost hobby hosting

Host small-scale server on a Raspberry Pi or an old PC and expose with IPNS-Link, readily, free of cost. No need to pay for a domain name. With built-in NAT-traversal, no need to buy public IP address either. With built-in [encryption](#security-and-trustlessness), no need to buy/manage SSL certificates.

### Implementations

[ipns-link](https://github.com/ipns-link/ipns-link)

[ipns-link-gateway](https://github.com/ipns-link/ipns-link-gateway)

### Contribute

Lots of things to be done, lots of help needed. You may contribute to this project in [multiple](https://github.com/ipns-link/contribute) ways.
