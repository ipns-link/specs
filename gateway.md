# IPNS-Link Gateway

This document specifies how the gateway operates. For illustration, the base-domain of our gateway is assumed to be `https://gateway.tld` located at IP `1.2.3.4`. Also, just as `CID` is Content ID for IPFS, by `KID` we shall refer to the Key ID for IPNS. This also includes DNSLinks.

### Types of incoming http-requests

In the [Bash implementation](https://github.com/ipns-link/ipns-link-gateway/blob/main/ipns-link-gateway), this is handled by the `handler` function.

##### https://gateway.tld/localPaths or https://www.gateway.tld/localPaths

GET or HEAD requests for resources local to the gateway server. For these, the gateway behaves as a simple file server. All other methods are forbidden with [`405 Method Not Allowed`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/405).

##### https://gateway.tld/ipfs/CID/* or https://www.gateway.tld/ipfs/CID/*

These requests are served as a subdomain-IPFS-Gateway, i.e. a [`303 See Other`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303) redirect is issued to https://CID.ipfs.gateway.tld/*. If a remote subdomain-IPFS-gateway is used for offloading, then that is where the redirect is targeted.

##### https://gateway.tld/ipns/KID/* or https://www.gateway.tld/ipns/KID/*

The `KID` is first transformed into base36 libp2p-key. If `KID` is a DNSLink that points to an IPFS path, then gateway behaves as subdomain-IPFS-gateway as described above. Otherwise, a [`307 Temporary Redirect`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307) is issued to https://KID.ipns.gateway.tld/* where `KID` is in base36.

##### https://CID.ipfs.gateway.tld/*

These requests are served as a subdomain-IPFS-Gateway. If a remote subdomain-IPFS-gateway, e.g. cf-ipfs.com, is used for offloading, then a 303 redirect is issued to https://CID.ipfs.cf-ipfs.com/*.

##### https://KID.ipns.gateway.tld/*

`KID` in this context must be a base36 libp2p-key or a DNSLink. This is the main type of URL that is directly handled by the IPNS-Link gateway. Discussed in detail below.

##### http://dnslink.tld/*

This type of requests may be formed when an IPNS KID is DNSLink'd to say `dnslink.tld`and the DNSLink is CNAME'd to `gateway.tld`. In this case, gateway first obtains the `KID` by resolving the DNSLink. Then, the behavior is same as that of https://KID.ipns.gateway.tld/*, described below.

##### http://1.2.3.4/*

307 redirect to https://gateway.tld/*

### How to handle https://KID.ipns.gateway.tld/path or http://dnslink-of-KID.tld/path

Ordered according to precedence. In the [Bash implementation](https://github.com/ipns-link/ipns-link-gateway/blob/main/ipns-link-gateway), this is handled by the `forward` function.

1. `path=/ipfs/CID/*` .AND. `Method==GET` : Same behavior as https://gateway.tld/ipfs/CID/* or https://www.gateway.tld/ipfs/CID/*, described above. **Exit**.
2. `path=/ipfs/KID/*`: 307 redirect to https://gateway.tld/path. **Exit**.
3. Transform `KID` to base36 libp2p key. This is not possible if KID is a DNSLink that points to an IPFS path. In that case, same behavior as https://CID.ipfs.gateway.tld/* and **Exit**. Otherwise, proceed with the base36 `KID`.
4. Check if `KID` is blocked. In case it is issue [`451 Unavailable For Legal Reasons`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/451) and **Exit**.
5. Access the JSON for `KID` from cache. If there is none in the cache or the cache is older than 24 hrs then:
   1. Retrieve IPNS records for `KID` and read the Manifest therefrom. If timed out waiting for IPNS record, try to access it from a well-connected public gateway such as `ipfs.io` that has IPNS-pubsub enabled. If IPNS record is found but no Manifest, behave as a subdomain-IPFS-gateway and **Exit**. Otherwise:
   2. Process Manifest as follows:
      1. If there is no IPNS-Link comment block in the Manifest, behave as a subdomain-IPFS-gateway and **Exit**. Otherwise:
      2. Read the `secret` corresponding to `KID` as provided locally by the gateway maintainer.
      3. Try to decipher the first line (ciphertext) of the comment block with a) the secret and b) the private-key. If successful, then cache the deciphered JSON against `KID`. Otherwise, pick a trusted-gateway randomly from the Manifest and 307 redirect to https://trusted-gateway.tld/ipns/`KID`/path and **Exit**. If no such gateway is provided in the Manifest, then [`401 Unauthorized`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401).
6. `Method==GET` .AND. `path=cache_path/*`: Serve the content of `/ipfs/cache_cid/*` and **Exit**.
7. Read the Listener's multiaddress from the cached JSON and try to connect to it. 
   - On successful connection, put it in the peering subsystem so that the connection is maintained indefinitely and retried automatically on disconnect.
   - On failure, assume cache is stale and try retrieve the Manifest. If connection still fails after that then 307 redirect to the `on_fail` URL and **Exit**. In case `on_fail` is empty, issue a [`504 Gateway Timeout`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/504) and **Exit**.
8. `Method==GET` .AND. `path=stream_path/*`: Serve the content of `/ipns/stream_kid/*` and **Exit**.
9. Setup a p2p-forward to the Listener-node for `KID` with protocol `/x/ipns-link/KID`, if not setup already. Forward the web-request to Origin through that.

### Pipeline

`Web-request` >> `Redirect` or `Manifest retrieval`  and `serve from IPFS if possible` >> `p2p-forward` (proxy)

