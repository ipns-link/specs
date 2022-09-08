# StarMap

Author: [@Winterhuman](https://github.com/Winterhuman)


## Intro

To further push the ability for anyone to host their site without cost, both to their freedoms and their wallets, Somajit invisioned a way people could obtain domain names of the form `domain.ipns`.

The problem is that creating a top-level domain isn't easy... or cheap... or something we as a small team can really manage. So instead, let's look at how we can convert our IPNS addresses into human-readable addresses that anyone can access, the goal is simple (in concept):

1. Allow people to access their exposed websites by a human-readable, memorable, and type-able address instead of the IPNS address each exposed Origin is given currently.
2. Allow anyone to navigate to these Origins without requiring custom DNS servers on the user's part.


## StarClusters

For the sake of simplicity, I'm going to be explaining the process backwards, from largest to smallest scale, in order to construct a progression of how each previous idea is needed for the next.

**StarClusters** are just PubSub rooms derived from an alphanumeric string, this allows you to search for a StarCluster by typing in human-readable names. Technically speaking, we could put all IPNS-Link nodes in one massive PubSub room named `.ipns` and call it a day, but, that means every search for a particular domain gets sent to everyone in the network which is very noisy. Instead, we'll break down StarClusters to refer to two parts of a **StarMap**.


## StarMaps


**StarMaps** are pseudo-domain names which allow you to attach an Origin to a human-readable name, they are self-assigned and not unique; people can have the same human-readable part and everything still works, here's an example StarMap:

`a7fjc... | k51origin... | ipns.link.ipns`


### Naming Section

`link` and `ipns` are each separate StarClusters, if you wanted to search for `ipns.link.ipns` you'd search in StarCluster `link` for nodes which are also subscribed to StarCluster `ipns`. This means you can do partial searches and find everyone within any one StarCluster. The second StarCluster, `ipns`, is optional and can be removed if desired.

As a side note, this system means `ipns.link.ipns` and `link.ipns.ipns` are the same since both are an overlap of StarClusters `ipns` and `link`, however, the StarMap will always favour the order the manifest specifies it in.

*Problem.* If we just used this, anyone could have the same human-readable name and they'd look identical, the ability to do partial searches means an attacker could easily find unique names and duplicate them to cause confusion. So, let's add two unique sections to the StarMap.


### Origin Section

`k51...` is an OriginID, think of it like a subdomain. Since the OriginID is derived from a keypair, it should be near impossible to imitate as another Origin, this means multiple `ipns.link.ipns` StarMaps can exist and each one would have a unique section to identify them with.

*Problem.* You couldn't just blindly trust that `*.ipns.link.ipns` will consist of only your Origins, anyone could join at any point either accidentially or intentionally, this makes it difficult to create a rule to trust all of your Origins and no one else. So, that's where the final part of the StarMap comes in.


### TOTP Section

This achieves a number of things:

1. TOTP means the Departure String, and thus it's encrypted counterpart, will change with each Manifest publish.
2. Each Origin's encrypted Departure String will be different even if the same TOTP number is underneath, it's impossible to tell if any two encrypted Departure Strings have the same TOTP number without decrypting them.
3. Only someone with the Departure Key can decrypt the encrypted Departure String and read the TOTP number, if two Origins have the same number at the same time then they share the same TOTP key.

The usage of TOTP and the OriginID prevents an attacker from pretending to be an Origin that's related to another one, copying `a7fjc...` into a new Manifest file won't work since the OriginID it encodes won't match the OriginID of the Origin who supposedly owns it thus invalidating it, this means the owner can trust `*.ipns.link.ipns` for as long as a verification process is used to automatically decrypt the Departure Strings and check the TOTP number before using the Origins.

Each Origin's config will allow you to define what keys are used, this allows you to adjust the behaviour:

1. (Default) Each Origin uses the same TOTP key and Departure Key, but as explained above, the encrypted Departure Strings yielded will still appear unrelated to each other until decrypted.
2. An Origin uses a separate TOTP key to prevent it from being associated with other Origins using the global TOTP key even when the Departure String is decrypted.
3. An Origin uses a separate Departure Key so multiple Departure Keys need to be known to compare it's Departure String against another Origins'.


### StarMap Format

Compiling all this information together, StarMaps are of the form:

`Encrypted Departure String | OriginID | StarCluster 2 (Optional) | StarCluster 1`

`k51...(optional).starcluster2 (optional).starcluster1.ipns`

The above configuration would look something like this in the Manifest:

```
<meta http-equiv="refresh" content="0; url=https://gateway.tld/ipns/OriginID">

<title>Redirecting...</title>

If you are not redirected automatically, follow this <a href='https://official-gateway.tld/ipns/OriginID'>link</a>

<!--IPNS-Link--
Name: ipns.link.ipns
EDS: a7fbi...  [EDS = Encrypted Departure String]
ciphertext in multibase (base64, inline)
https://trusted-gateway1.tld
https://trusted-gateway2.tld
--IPNS-Link-->
```


## Additional

Public gateways could cache StarMaps to speedup searches and reduce the amount of traffic sent to StarClusters.

A browser extension, where you can import the Departure Key to automatically read the TOTP numbers of your Origins on public gateways, may be desirable. Or some form of secure way public gateways can let you enter the Departure Key without the gateway knowing the key you entered.


## Conclusion


### Advantages

- Anyone can have any name they want and even share the name with others.
- You can identify which Origins are yours, even from identically named StarMaps, no one can easily imitate being another Origin since they need to bypass some combination of: IPNS keypairs, TOTP keys, and Departure Keys.
- You don't need to know the full name of a StarMap to find it, you can navigate and search through the network.
- Any node capable of searching the StarClusters, reading the Manifest files, and decrypting the encrypted Departure Strings could resolve StarMaps themselves without needing any public gateways.
- Listener and Publisher nodes don't need to be any more active then they already are.


### Disadvantages

- It's not as simple as entering `domain.tld` and trusting that it's going to the right place; you need a verification system to decrypt and check the TOTP number just to verify that it's your Origin. A browser extension, or secure means of entering it into a public gateway, will be needed to enhance usability.
