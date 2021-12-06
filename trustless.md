# How to build a trustless platform

With IPNS-Link, Gateway is always the man in the middle (MITM) between Browser and Origin. Usually, TLS (SSL) terminates at Gateway for good - so that Gateway can serve some content from IPFS wherever possible, improving latency and user-experience, while also offloading Origin. The decentralized nature of the system lets anyone host a public Gateway, which introduces the possibility of malicious Gateways. Although such a threat is somewhat mitigated by the ability of Origin to be accessible only through its trusted Gateways, even the trusted Gateways might become rogue, if hijacked. These would be non-issues if end-users would simply install our open-source code and run their own Gateways locally. But that is impractical - it's safe to assume most wouldn't do that.

Technically, Listener too is a middle-man, but being local to the Origin, it is nothing to be worried of.

### Solution: TLS passthrough with effective MITM monitoring

In modern TLS, during the initial handshake, the client transmits the fully qualified domain name (FQDN) of its desired server, using a Server Name Indication (SNI) field. Because this information is transmitted before the client even knows the server's public key, SNI is unencrypted. A [TLS Passthrough](https://github.com/SomajitDey/isp) (TLSP) is an MITM that can read the SNI, and route the traffic accordingly to the desired backend, which terminates TLS.

But there is nothing to prevent the TLSP from acting as an [MITM proxy](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/#the-mitm-in-mitmproxy) (MITMP), terminating the TLS itself and creating a parallel TLS connection with the backend. Essentially, an MITMP acts as the server to the client and as the client to the server.

Although MITMPs can't be prevented altogether, they can be caught. For any reputed Gateway provider, the risk of getting caught is sufficient to preclude MITM attacks, as those who do get caught, may be blacklisted and banned.

Before adopting TLSP therefore, one needs a feasible MITM monitoring system. Once such a system is in place, Gateways may offer an optional TLSP mode to make Origin <--> Browser connection truly end-to-end encrypted (E2E).

### MITM Monitor

Catching an MITM proxy from the client-side relies on the fact that the proxy cannot provide the same TLS certificate (public key) as the Origin, if it is to decrypt the traffic. Note that Origin can screen for MITMP by opening a TLS tunnel to itself through the suspected Gateway and comparing the received TLS certificate with its own. However, if Origin has a static public IP, the Gateway, to avoid getting caught, can simply passthrough those requests that originate from Origin's IP. Such screening, therefore, is not robust. Also, there must be a way to prove to any end-user that any claimed E2E route to any Origin through any given Gateway, is free from MITMP.

### E2E protocol and client-side monitor

The default mode of IPNS-Link lets Gateway inspect the HTTP headers in order to serve content efficiently. The FQDN in this mode reads `kid.ipns.gateway.tld`, where `kid` denotes the IPNS key of Origin. To distinguish the E2E mode from this, the corresponding FQDN would be `kid.e2e.gateway.tld`. To terminate TLS, Origin needs a keypair and a valid TLS certificate. It might use its own libp2p keypair (`kid`), provided it is RSA 2048. Otherwise, it can create an RSA 2048 keypair and publish the fingerprint (SHA256) of the public key with its Manifest, in order to link it with its `kid`.

The following describes how E2E is established.

1. Gateway receives an HTTPS request with SNI `kid.e2e.gateway.tld`. It extracts the `kid` from the SNI.
2. Gateway retrieves the Manifest for `kid`, decrypts it, peers with Origin and establishes TLS passthrough using the p2p protocol `/x/e2e/kid`. It also reads the TLS public key fingerprint of Origin, from the Manifest.
3. Origin creates a CSR corresponding to the SNI, using its RSA 2048 public key. Using ACME, it gets a valid TLS certificate from Let's Encrypt or ZeroSSL. To complete the TLS handshake, it responds to the Client Hello from Browser with the certificate, in its Server Hello.
4. While passing the Server Hello through, Gateway extracts the public key and compares it with the fingerprint it read from the Manifest. On match, it lets the passthrough proceed. Otherwise, further connectivity is aborted, and the Origin is blacklisted locally. This prevents Origin from linking one public key in the Manifest and using another for TLS.

The following describes how to monitor for possible MITMP at Gateway using client-side code. The client-side code may consist of a browser extension that reads URLs of the form `https://kid.e2e.gateway.tld/*` and screens `gateway.tld` for MITMP, or, it may consist of a static web-page (hosted on IPFS) where the end-user can manually enter `gateway.tld` and `kid` to screen for MITMP. The monitor might also be a command-line script or desktop app.

1. Start a TLS handshake with `https://kid.e2e.gateway.tld` and extract the certificate from server hello. Therefrom extract the public key.
2. Retrieve Origin's fingerprint from its Manifest using any IPFS-gateway.
3. Compare the public key with the fingerprint. If they match, there is no MITMP. Otherwise warn the user and report the Gateway.

**Note**: 

- Once CAs start supporting ED25519, the libp2p-key of the Origin may be used for TLS. The `kid` itself would serve as the fingerprint then.
- Client-side monitoring from different clients originate from different IP addresses - unpredictable to the Gateway. The suspect Gateway can no more dupe the monitor by passing TLS through instead of proxying.
- Libp2p tunnels are themselves secured with [TLS 1.3](https://github.com/libp2p/specs/blob/master/tls/tls.md).

