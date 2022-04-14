# How to build a trustless platform

With IPNS-Link, Gateway is always the man in the middle (MITM) between Browser and Origin. Usually, TLS (SSL) terminates at Gateway for good - so that Gateway can serve some content from IPFS wherever possible, improving latency and user-experience, while also offloading Origin. The decentralized nature of the system lets anyone host a public Gateway, which introduces the possibility of malicious Gateways. Although such a threat is somewhat mitigated by the ability of Origin to be accessible only through its trusted Gateways, even the trusted Gateways might become rogue, if hijacked. These would be non-issues if end-users would simply install our open-source code and run their own Gateways locally. But that is impractical - it's safe to assume most wouldn't do that.

Technically, Listener too is a middle-man, but being local to the Origin, it is nothing to be worried of.

### Solution: TLS passthrough with effective MITM monitoring

In modern TLS, during the [initial handshake](https://www.cloudflare.com/en-in/learning/ssl/what-happens-in-a-tls-handshake/), the client transmits the fully qualified domain name (FQDN) of its desired server, using a Server Name Indication (SNI) field. Because this information is transmitted before the client even knows the server's public key, SNI is unencrypted. A [TLS Passthrough](https://github.com/SomajitDey/isp) (TLSP) is an MITM that can read the SNI, and route the traffic accordingly to the desired backend, which terminates TLS.

But there is nothing to prevent the TLSP from acting as an [MITM proxy](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/#the-mitm-in-mitmproxy) (MITMP), terminating the TLS itself and creating a parallel TLS connection with the backend. Essentially, an MITMP purports to be the server to the client and the client to the server.

Although MITMPs can't be prevented altogether, they can be caught. For any reputed Gateway provider, the risk of getting caught is sufficient to preclude MITM attacks, as those who do get caught, may be blacklisted and banned.

Before adopting TLSP therefore, one needs a feasible MITM monitoring system. Once such a system is in place, Gateways may offer an optional TLSP mode to make Origin <--> Browser connection truly end-to-end encrypted (E2E).

### MITM Monitor

Catching an MITM proxy from the client-side relies on the fact that the proxy cannot provide the same TLS certificate (public key) as the Origin, if it is to decrypt the traffic. Note that Origin can screen for MITMP by opening a TLS tunnel to itself through the suspected Gateway and comparing the received TLS certificate with its own. However, if Origin has a static public IP, the Gateway, to avoid getting caught, can simply passthrough those requests that originate from Origin's IP. Such screening, therefore, is not robust. Also, there must be a way to prove to any end-user that any claimed E2E route to any Origin through any given Gateway, is free from MITMP.

### E2E protocol and client-side monitor

The default mode of IPNS-Link lets Gateway inspect the HTTP headers in order to serve content efficiently. The FQDN in this mode reads `oid.ipns.gateway.tld`, where `oid` denotes the OriginID. To distinguish the E2E mode from this, the corresponding FQDN would be `oid.e2e.gateway.tld`. To terminate TLS, Origin needs a keypair and a valid TLS certificate. It might use its own libp2p keypair (`oid`), provided it is RSA 2048. Otherwise, it can create an RSA 2048 keypair and publish the fingerprint (SHA256) of the public key with its Manifest, in order to link it with its `oid`.

The following describes how E2E is established.

1. Gateway receives an HTTPS request with SNI `oid.e2e.gateway.tld`. It extracts the `oid` from the SNI.
2. Gateway retrieves the Manifest for `oid`, decrypts it, peers with Origin and establishes TLS passthrough using the p2p protocol `/x/e2e/oid`. It also reads the TLS public key fingerprint of Origin, from the Manifest.
3. Origin creates a [CSR](https://en.wikipedia.org/wiki/Certificate_signing_request) corresponding to the SNI, using its RSA 2048 public key. Using ACME, it gets a valid TLS certificate from Let's Encrypt or ZeroSSL. To complete the TLS handshake, it responds to the Client Hello from Browser with the certificate, in its Server Hello.
4. While passing the Server Hello through, Gateway extracts the public key and compares it with the fingerprint it read from the Manifest. On match, it lets the passthrough proceed. Otherwise, further connectivity is aborted, and the Origin is blacklisted locally. This prevents Origin from linking one public key in the Manifest and using another for TLS.

The following describes how to monitor for possible MITMP at Gateway using client-side code. The client-side code may consist of a browser extension that reads URLs of the form `https://oid.e2e.gateway.tld/*` and screens `gateway.tld` for MITMP, or, it may consist of a static web-page (hosted on IPFS) where the end-user can manually enter `gateway.tld` and `oid` to screen for MITMP. The monitor might also be a command-line script or desktop app.

1. Start a TLS handshake with `https://oid.e2e.gateway.tld` and extract the certificate from server hello. Therefrom extract the public key.
2. Retrieve Origin's fingerprint from its Manifest using any IPFS-gateway.
3. Compare the public key with the fingerprint. If they match, there is no MITMP. Otherwise warn the user and report the Gateway.

**Note**: 

- Client-side monitors screen for any on-path MITMP even at the level of ISP or DNS provider.
- Once CAs start supporting ED25519, the libp2p-key of the Origin may be used for TLS. The `oid` itself would serve as the fingerprint then.
- Client-side monitoring from different clients originate from different IP addresses - unpredictable to the Gateway. The suspect Gateway can no more dupe the monitor by passing TLS through instead of proxying.
- The [TLS-ALPN-01](https://letsencrypt.org/docs/challenge-types/#tls-alpn-01) challenge for domain validation cannot always screen for MITMP. To avoid getting detected, the Gateway can simply passthrough the validation traffic as soon as it sees "[acme-tls/1](https://datatracker.ietf.org/doc/html/rfc8737#section-4)" as the Application-layer protocol.
- Libp2p tunnels are themselves secured with [TLS 1.3](https://github.com/libp2p/specs/blob/master/tls/tls.md).

### HTTPS enabled DNSLinks and DNS-based load balancing

[DNSLink](https://dnslink.io/)s are most convenient when they are CNAMEd to a target Gateway. In conventional mode where TLS terminates at the Gateway, one cannot use https://dns.link unless the target Gateway can generate a TLS certificate for `dns.link` on the fly using [non-DNS-based challenges](https://letsencrypt.org/docs/challenge-types/). Because of the huge number of possible DNSLinks, which cannot be served with a catch-all wildcard certificate, most Gateways may not be able to provision per DNSLink certificates, because of latency and disk space concerns.

Any given Origin, however, would be associated with only a small number of DNSLinks, and therefore, can provision certificates with ease. Whenever a TLSP Gateway receives HTTPS traffic with `dns.link` as the SNI, it can retrieve the `oid` by resolving the DNSLink and route the traffic accordingly.

Because TLS is terminated by the Origin, and *all* Gateways forward to the Origin, `https://dns.link` would work regardless of which Gateway `dns.link` is CNAMEd to. More conveniently, DNS records for `dns.link` may be configured to target multiple Gateways, in order to take advantage of [DNS-based load balancing](https://www.cloudflare.com/en-in/learning/performance/what-is-dns-load-balancing/).

### Umbrella domain

Conventional DNSLinks, however, are not feasible for self-hosters who cannot afford a domain name. In view of this, IPNS-Link organization might provide an overarching domain, such as `umbrella.tld`, as follows:

1. IPNS-Link organization is the sole controller of the domain. For DNS-based load balancing, it CNAMEs `*.umbrella.tld` to all registered public Gateways that it actively monitors. Whenever a Gateway fails the health check, the corresponding CNAME or A record is removed, until it is online again.
2. Whenever a Gateway receives `https://oid.umbrella.tld`, it forwards the request to the Origin corresponding to `oid`, which generates a certificate for `oid.umbrella.tld` on the fly using a TLS-ALPN-01 challenge. Note that the same certificate can now be used for requests coming via different Gateways.

Client-side MITM monitor might also benefit from this. Repeated monitoring attempts for `oid.umbrella.tld` screens all known paths to Origin.

Having an umbrella domain can enable DNS-based load-balancing for the default (non-E2E) mode too. To illustrate, if `umbrella.tld` is CNAMEd to all the registered Gateways, then URLs like `https://umbrella.tld/ipns/oid` may be TLS-terminated at the Gateway and redirected to `https://oid.ipns.gateway.tld`.

