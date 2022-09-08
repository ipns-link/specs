# IPNS-Link New (V3)

## Intro

To improve on IPNS-Link's current spec, and take advantage of new features in IPFS, the spec should be changed to reduce resource consumption, decrease code complexity, and increase decentralisation.

## New Spec

### Manifest

Instead of inlining a JSON file into the IPNS record's `Value` field, custom fields can be added to IPNS records which state all the necessary information. Here's what it might look like inside the CBOR `IpnsEntry.data` object:

```
...

required bytes host = "Host header value"
required bytes peerid = "Exposer PeerID"
required bytes route {
	"/ip4/1.2.3.4/tcp/567",
	"/ip6/1:2:3:4::/tcp/567"
}
optional bytes on_fail = "CID of fallback webpage"
```

- `host`: See **Origin Config**.
- `peerid`: Defines the PeerID of the Exposer, the value must be a valid PeerID string.
- `route`: Defines the truncated multiaddresses of the Exposer. The value of `peerid` is suffixed onto the multiaddresses as `.../p2p/peerid`.
- `on_fail`: See **Origin Config**.

The IPNS record containing these fields must **not** exceed 2MiB, otherwise the IPNS record will be rejected by IPFS nodes. The IPNS address associated with IPNS-Link IPNS records is called the **OriginID**.

Additionally, it may be advantageous to create a parent field (`IpnsEntry.data.ipns-link`) which contains the other custom fields. This would ensure the field names are not accidentally shared with other applications.

### Exposer

Instead of utilising two separate IPFS nodes for publishing and exposing, we'll call this the Publisher-Exposer model, we can instead shift to a single node with full DHT access (`DHTClient` specifically) just called the **Exposer**.

Rather than granting access to origins through the knowledge of the Exposer's multiaddresses, each origin will instead have an allow list of PeerIDs who can connect to it's multistream protocol. This is a much stronger method of security, and gets around needing to hide the Exposer from the DHT.

To reduce bandwidth consumption, the ResourceManager will instead enforce the maximum amount of bandwidth that can be used. This allows for more fine grain control of resource usage. Additionally, not needing to constantly restart a Publisher node means IPNS-Link will be less reliant on bootstrap nodes.

For a given origin at `127.0.0.1:80`, with an OriginID of `k51origin`, the Exposer will create a multistream protocol named `/origin/k51origin` which can access it. Only Libp2p/IPFS nodes that are trusted for that origin can connect to the protocol, as defined in the origin's config.

#### Origin Config

Here's an example config:

```
origin: "127.0.0.1:80"
host: "127.0.0.1:80"
trusted: {
	"12D3KooWgateway1...",
	"12D3KooWgateway2..."
}
on_fail: "QmFallback..."
```

- `origin`: Defines the IP:port you want to expose. Can be an origin or a reverse proxy in front of multiple origins.
- `host`: Defines the value to be set for the HTTP `Host:` header. If undefined, the value of `origin` will be used.
- `trusted`: Defines a list of PeerIDs belonging to trusted IPNS-Link gateways you want to be able to access the origin.
- `on_fail`: Defines a CID or IPNS address for a webpage, the user will be shown this page if the Exposer is inaccessible. If undefined, the gateway will instead present it's own error page.

**WARNING!** All origins behind the same reverse proxy **must** share the same set of `trusted` gateways between them. If `bad-gateway.tld` had access to `/origin/origin1`, but wanted to access the service behind `/origin/origin2`, it could just send HTTP packets to `/origin/origin1` with `Host: "Origin2 host value"` to access it.

### Gateway

IPNS-Link gateways will now consist of an internal Libp2p node with no routing, rather than an IPFS node with full DHT access. This is because:

- Gateways can fetch IPNS records from public (or local) IPFS gateways using CAR (once https://github.com/ipfs/specs/issues/320 is accepted and implemented) which can be verified trustlessly.
- IPNS-Link IPNS records already contain the multiaddresses of the Exposer within them, meaning no routing is required to search for it's PeerID.
- And ultimately, the only function left for the gateway's node is to connect to the Exposer, which is purely Libp2p.

The advantage of making IPNS-Link gateways "light IPFS clients" is that:

- Gateway software is significantly less complex, most of it's functionality focuses on HTTP which is well established and documented.
- The Libp2p node doesn't need routing, which reduces the background bandwidth usage (and it doesn't significantly decrease decentralisation as bootstrap nodes and public IPFS gateways aren't that different in this case).
- And despite the above, CAR files guarantee that the fetched records haven't been tampered with thus ensuring the system is trustless from the gateway's point of view.

> IPNS-Link gateway software may use an IPFS node if a "full" IPFS client is desired.

#### Gateway Logic

> See https://github.com/ipfs/specs/blob/main/http-gateways/PATH_GATEWAY.md#http-response for standard IPFS gateway HTTP logic.

##### IPFS Paths & Subdomains

If `gateway.tld/ipfs/cid` or `cid.ipfs.gateway.tld` is given, `307` (Temporary Redirect) redirect to the equivalent https://dweb.link URL (other public gateways may be used).

##### IPNS Paths

If `gateway.tld/ipns/key` is given, `301` (Moved Permanently) redirect to `key.ipns.gateway.tld`.

Or, if `key` is not a subdomain-compatible CID, return error code `400` (Bad Request).

##### IPNS Subdomains

If `key.ipns.gateway.tld` is given, request `key.ipns.dweb.link/?format=car`, and return any error code given by the public gateway.

If `key.ipns.gateway.tld` is fetched successfully, but `IpnsEntry.data.ipns-link` or it's required children are missing, return error code `502` (Invalid Response).

If `key.ipns.gateway.tld` is fetched successfully, and `IpnsEntry.data.ipns-link` & it's required children are present, but the IPNS record could **not** be verified, return error code `502` (Invalid Response).

If `key.ipns.gateway.tld` is fetched successfully, and `IpnsEntry.data.ipns-link` & it's required children are present, and the IPNS record **was** verified, extract the values.

If valid multiaddresses can **not** be created from `IpnsEntry.data.ipns-link.{peerid,route}`, return error code `502` (Invalid Response).

If valid multiaddresses **could** be created from `IpnsEntry.data.ipns-link.{peerid,route}`, pass them to the internal Libp2p/IPFS node.

##### Internal Node

If the Exposer can **not** be connected to, return error code `503` (Unavailable).

If the Exposer **can** be connected to, but `/origin/key` does **not** exist, return error code `500` (Invalid Response).

If the Exposer **can** be connected to, `/origin/key` **does** exist, but access is refused, return error code `403` (Forbidden).

If the Exposer **can** be connected to, and `/origin/key` **can** be accessed, serve the exposed webpage at `key.ipns.gateway.tld`.

