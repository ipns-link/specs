# Static Manifest Gateways

Author: @Winterhuman

## Issues with Public IPNS-Link Gateways

- Public Gateways require a constantly running server.
- They require a domain name to be publicly accessible.
- They require the user to trust the Gateway's actions.


## Idea

**Static Manifest Gateways** take advantage of Manifest files, Manifests are just HTML files which can be fetched over IPFS, so combining Manifests with Javascript and ServiceWorkers can give users publicly accessible Gateways which are serverless. 

Since Static Gateways are actually Manifests, they already contain the necessary multiaddresses of the Listener, we can then use a js-ipfs node to connect to the Listener and access the Origin.

Here's a simple diagram to explain how this works:

```
1. Browser -> 'oid.ipns.ipfs-gateway.tld' <-> IPFS <-> Publisher & Neighbouring Nodes
2. Browser -> ServiceWorker + js-ipfs <-> Listener <-> Origin
```

1. First, the Browser navigates to `oid.ipns.ipfs-gateway.tld`.
2. The IPFS Gateway fetches Manifest from Publisher or it's neighbours.
3. The Manifest then sources a Javascript file from IPFS, let's call it the **Gateway Script**.
4. The Gateway Script creates a ServiceWorker and refreshes the page, essentially replacing the Manifest with what it wants to serve.
5. A login page appears prompting the user to enter a password to decrypt the Manifest's ciphertext before refreshing again.
6. The ServiceWorker then starts a js-ipfs node which connects to the Listener with the Manifest's multiaddresses.
7. The ServiceWorker takes the HTTP output from the Origin and feeds it to the Browser, thus rendering the Origin as if it was accessed directly.
8. The user can now access and interact with the Origin.

Instead of encrypting the JSON with GPG and the public keys of gateways, you instead encrypt it with a user-defined password to be entered in the login page, this prevents random people from navigating to a Manifest and getting instant access to the Origin tied to it.


# Summary


## Advantages

- Static Manifest Gateways can be fetched by Public IPFS Gateways for maximum accessibility, or they can be fetched by locally running IPFS nodes for maximum privacy. When IPFS nodes are common in Browsers then users can benefit from both advantages.
- IPFS Gateway's can't see what the ServiceWorker presents to the user after the Manifest is first loaded, the connection between the js-ipfs node and Origin is also encrypted outside of the HTTPS connection.
- IPFS Gateway's won't suffer the bandwidth from the connection since the js-ipfs connection is peer-to-peer.
- They are extremely difficult to block or censor, all it takes is a one byte change and now the Static Manifest Gateway has a new CID.
- The Browser doesn't need any modification or extension to allow Static Manifest Gateways to work.
- Manifest's can stay minimal by sourcing the Javascript files from separate files (Ideally from IPFS when possible).
- Manifest's can be themed per the user's liking to create pre-home, login-like pages. (Maybe **Static Exposer Gateways** could be explored?)


## Disadvantages

- `oid.ipns.gateway.tld/images/image1` isn't possible, those request go to the IPFS Gateway, IPFS URLs must be used as a substitute.
- Malicious Public IPFS Gateways can replace the Manifest with a malicious file, users need to trust the IPFS Gateway they access the Manifest over.

## Possible Improvements

- Create Static Exposer Gateways where the user can choose which Origin they want to access.
- Run a Delegated Router at the Origin for the js-ipfs node to use in case IPFS relative URLs can't be resolved in Browser or locally.
