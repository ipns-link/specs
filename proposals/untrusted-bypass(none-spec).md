# Untrusted Gateway Bypass

In Manifests, we use encryption to hide the multiaddresses of the Listener from untrusted Gateways, this is to prevent malicious Gateways from spamming the Listener, however, it turns out that although the multiaddresses are hidden there's still a way an untrusted Gateway can access the Origin:

1. User goes to `gateway.untrusted.tld` and searches for `oid`.
2. The untrusted Gateway resolved the `oid` to find a Manifest with ciphertext it can't decrypt.
3. The untrusted Gateway makes a request to `gateway.trusted.tld` as listed in the Manifest, the trusted Gateway then resolves the `oid` and provides the Origin at `oid.ipns.gateway.trusted.tld`.
4. The untrusted Gateway proxies `oid.ipns.gateway.trusted.tld` for the user at `oid.ipns.gateway.untrusted.tld`, the user will have no idea that the untrusted Gateway didn't decrypt the ciphertext itself thus negating the purpose of the encryption.

This vulnerability isn't as bad as it first seems though, and, it's also completely unavoidable:

1. The untrusted Gateway still has no idea what the multiaddresses of the Listener are, it relies on the trusted Gateway to do the proxying, this means the untrusted Gateway would still be subject to rate limiting if it tried to spam the Origin through the trusted Gateway.
2. Users should never access an untrusted Gateway in the first place, there's no reason why they'd use a Gateway other than one defined in the trusted Gateway list of their Origin, we can't control user behaviour.
