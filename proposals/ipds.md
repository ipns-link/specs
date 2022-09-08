# IPDS

Author: [@gxnt-samir](https://github.com/gxnt-samir), [@Winterhuman](https://github.com/Winterhuman)

**IPDS** (Interplanetary Domain System) is a way to share a DNS-like database between **Authoritative Nodes** who are part of a PubSub room, nodes in the room will sync the IPDS database between themselves and an extension will point to the PubSub room to resolve domain names.

*Here's how it would work:*


## IPDS Database

IPDS uses a CRDT to prevent conflicts and consists of records in the form:

`IPNS of Service/IPDS Database | domain.tld`

If a record lacks a signature, or that signature cannot be verified by any trusted public keys, then the browser extension will ignore it.

IPNS addresses don't just have to be for a service, the IPNS address can point to the domain owner's personal IPDS database which contains the actual domains and subdomains they own, this means the owner only has one IPDS record to sign. The owner's personal IPDS records do not need to be signed since the record pointing to the database already is.


## Authoritative Nodes

**Authoritative Nodes** [ANs] are similar to authoritative DNS servers and maintain the IPDS database, they are connected together by a shared PubSub room, this distributes the load between all nodes in the PubSub room.

ANs sync IPDS records between each other so that new records can propagate throughout the room, records that ANs create will be signed by that ANs private key(s), ANs will advertise their public key(s) so that a record's signature can be verified.


## Overseers

**Overseers** are ANs that create the PubSub rooms, ANs can only subscribe to one Overseer PubSub room at a time to prevent record conflicts between Overseers, ANs will only add records to their database if they're signed by the Overseer.

```
Overseer 1's PubSub room
  \_ Authoritative Node 1
  |_ Authoritative Node 2
  |_ ...
```

In the browser extension, you can define multiple PubSub rooms/Overseers in a particular order. In the event that multiple Overseer's rooms say that some domain `domain.tld` resolves to multiple different IPNS addresses, the PubSub room/Overseer at the top of the list will be trusted and it's record will be treated as the correct one amongst them.


## IPDS Database Types

There are two databases for IPDS:

- Overseer IPDS Database [OID]
- AN IPDS Database [AID]

The **OID** is for records signed and created by the Overseer of the PubSub room, these records are shown to the browser extension by default.

The **AID** is for records an AN has personally signed and created for themselves, these records are usually for the AN owner's own use. This allows ANs to create their own records or even override domains within OID records to point to other IPNS addresses, however, this only works if the browser extension is set to allow AID records from that particular AN.
