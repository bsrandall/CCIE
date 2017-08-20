### IPSec Overview
- IPSec is a suite of protocols overseen by the IETF
- IPSec offers 4 services:
  - access-control
  - authentication
  - confidentiality
  - integrity

The baseline architecture and fundemental components of IPSec are defined as the following:
- **Security Protocols:** Authentication Header (AH) and Encapsulation Security Payload (ESP)
- **Key Management:** ISAKMP, IKE
- **Algorithms:** for encryption and authentication

#### Terminology
*Encryption* is the transformation of plain text into a form that makes the original text incomprehensible to an unauthorized recipient that does not hold a matching key to decode or decrypt the encrypted message.

*Decryption* is the reverse of encryption; it is the transformation of encrypted data back into plain text.

A *cryptographic algorithm*, also called a *cipher*, is the mathematical function used for encryption and decryption. Security of data in modern cryptographic algorithms is based on the key (or keys). It doesn't matter if an eavesdropper knows your algorithm; if he or she doesn't know your particular key, an eavesdropper will be unable read your messages.

Cryptographic Algorithms can be classified into two categories, Symmetric and Asymmetric.

Symmetric cryptographic algorithms are based on the sender and receiver of the message knowing and using the same secret key. The sender uses a secret key to encrypt the message, and the receiver uses the same key to decrypt it. The main problem with using the symmetric key approach is finding a way to distribute the key without anyone else finding it out. Anyone who overhears or intercepts the key in transit can later read and modify messages encrypted or authenticated using that key, and can forge new messages. DES, 3DES, and AES are popular symmetric encryption algorithms.

Asymmetrical encryption algorithms, also known as public key algorithms, use separate keysone for encryption and another for decryption. The encryption key is called the public key and can be made public. Only the private key, used for decryption, needs to be kept secret. Although the public and private keys are mathematically related, it is not feasible to derive one from the other. Anyone with a recipient's public key can encrypt a message, but the message can only be decrypted with a private key that only the recipient knows. Therefore, a secure communication channel to transmit the secret key is no longer required as in the case of symmetric algorithms.

In reality, public key encryption is rarely used to encrypt messages because it is much slower than symmetric encryption; however, public key encryption is used to solve the problem of key distribution for symmetric key algorithms, which is, in turn used to encrypt actual messages. Therefore, public key encryption is not meant to replace symmetric encryption, but can supplement it and make it more secure.

Encrypting a message with a private key creates a *digital signature*, which is an electronic means of authentication and provides *non-repudiation*. Non-repudiation means that the sender will not be able to deny that he or she sent the message. That is, a digital signature attests not only to the contents of a message, but also to the identity of the sender. Because it is usually inefficient to encrypt an actual message for authentication, a document hash known as a message digest is used. The basic idea behind a message digest is to take a variable length message and convert it into a fixed length compressed output called the message digest. Because the original message cannot be reconstructed from the message digest, the hash is labeled "one-way."

#### IPSec Security Protocols

##### IPSec Transport Mode
n transport mode, an IPSec header (AH or ESP) is inserted between the IP header and the upper layer protocol header. The IP header is the same as that of the original IP packet except for the IP protocol field, which is changed to ESP (50) or AH (51), and the IP header checksum, which is recalculated. IPSec assumes the IP endpoints are reachable. In this mode, the destination IP address in the IP header is not changed by the source IPSec endpoint; therefore, this mode can only be used to protect packets in scenarios in which the IP endpoints and the IPSec endpoints are the same. In other words, this only works for host-to-host communication. Another limitation of transport mode is that it cannot be used with NAT translation of packets between IPSec peers. Also, for most hardware encryption engines, it is less efficient to encrypt transport mode than tunnel mode, because transport mode requires displacement of the IP header to make room for the ESP or AH header.

##### IPSec Tunnel Mode
IPSec VPN service using transport mode and GRE encapsulation between the VPN gateways at each site is a very popular option for site-to-site VPNs. But what about an IP node that has no GRE support, yet requires the establishment IPSec VPN connectivity with another site? The most common example of this is a telecommuter. IPSec tunnel mode helps address this situation.

In tunnel mode, the original IP packet is encapsulated in another IP datagram, and an IPSec header (AH or ESP) is inserted between the outer and inner headers. Because of this encapsulation with an "outer" IP packet, tunnel mode can be used to provide security services between sites on behalf of IP nodes behind the gateway router at each site. Also, this mode can be used for the telecommuter scenario of connecting from an end host to an IPSec gateway at a site.

##### Encapsulating Security Header (ESP)
ESP provides confidentiality, data integrity, and optional data origin authentication and anti-replay services. It provides these services by encrypting the original payload and encapsulating the packet between a header and a trailer. ESP is identified by a value of 50 in the IP header. The ESP header is inserted after the IP header and before the upper layer protocol header. The IP header itself could be a new IP header in tunnel mode or the original IP packet's header in transport mode.

{% include image.html file="ESP-protection.png" %}

The security parameter index (SPI) in the ESP header is a 32-bit value that, combined with the destination address and protocol in the preceding IP header, identifies the security association (SA) to be used to process the packet. The SPI is an arbitrary number chosen by the destination peer during Internet Key Exchange (IKE) negotiation between the peers. It functions like an index number that can be used to look up the SA in the security association database (SADB).

The sequence number is a unique monotonically increasing number inserted into the header by the sender. Sequence numbers, along with the sliding receive window, provide anti-replay services. The anti-replay protection scheme is common to both ESP and AH.

The data being protected (or, more specifically, being encrypted by ESP) is in the payload data field. The algorithm used to encrypt the payload may require an initialization vector (IV), which is also carried in the data payload. Note that the IV is authenticated but not encrypted. If the encryption algorithm used is DES, then the first eight bytes of the protected data field is the IV; 3DES and AES also have an 8-byte IV.

Padding in the ESP header is the addition of bits to the ESP header; the number of bits to be padded depends on the encryption algorithm that is used. The Pad Length field indicates the number of pad bytes added so that the original data can be restored on decryption.

Authentication digest in the ESP header is used to verify data integrity. Because authentication is always applied after encryption, a check for validity of the data is done upon receipt of the packet and before decryption.

##### Authentication Header (AH)
AH provides connectionless integrity, data authentication, and optional replay protection but, unlike ESP, it does not provide confidentiality. Consequently, it has a much simpler header than ESP. AH is an IP protocol, identified by a value of 51 in the IP header. The Next header field indicates what follows the AH header. In transport mode, it will be the value of the upper layer protocol being protected (for example, UDP or TCP). In tunnel mode, this value is 4.

AH in transport mode is useful if the communication endpoints are also the IPSec endpoints. In tunnel mode, AH encapsulates the IP packet and an additional IP header is added before the AH header. Although the tunnel mode of AH could be used to provide IPSec VPN end-to-end security, there is no data confidentiality in AH and hence this mode is not too useful.

The SPI and sequence numbers have the same significance as in ESP. The authentication digest has one key difference from ESP: With AH, authentication is provided to the IP header in addition to the payload. As AH creates the authentication data on the entire packet, including the IP header, some of the IP fields will change in transit; therefore, all those fields in the IP header that may change in transit are zeroed out before the authentication digest is hashed. The fields that zero out include type of service (ToS) bits, flags, fragment offset, time-to-live (TTL), and header checksum. These fields are zeroed out because authenticating a changed value in transit (for example, TTL) will cause the authentication hash to have a mismatch from the sender and the packet will be dropped.

#### Key Management and Security Associations
Both 3DES and AES are symmetric algorithms and therefore have to deal with secure key distribution. Problems arise when this key distribution has to occur over a public network, such as the internet. Collectively, the generation, distribution, and storage of keys is called key management. All crypto systems must deal with key management issues. The default IPSec method for secure key negotiation is the Internet Key Exchange (IKE) protocol. IKE is designed to provide mutual authentication of systems, as well as to establish a shared secret key to create IPSec security associations. Before delving into how IKE works, it may be helpful to review the Diffie-Hellman key management protocol that is used by IKE to exchange a secret key over an insecure medium (such as the Internet).

##### The Diffie-Hellman Key Exchange

First Alice and Bob agree publicly on a prime number and a modulus (we will use 7 and 4). Then Alice selects a private randum number, say 3, and calculates $7^3$ mod 3 = 1 and sends it publicly to Bob. Then Bob selects his private random number, say 2, and calculates $7^2$ mod 3 = 1 and sends it publicly to Alice. Alice then takes Bob's public result and raises it to the power of her private number mod the original modulus. So she takes $1^3$ mod 3 = 1 so 1 is the shared secret. Computing the same for Bob, he takes Alice's public number (1) and raises it to the power of his private number mod the original modulus. So he takes $1^2$ mod 3 = 1. This is the same as Alice's shared secret and it has been computed without Eve being able to compute it.

##### Security Associations and IKE
A security association, more commonly referred to as an SA, is a basic building block of IPSec. An SA is an entry in the SA database (SADB), which contains information about the security that has been negotiated between two parties for IKE or IPSec. There are two types of SAs:
- IKE or ISAKMP SA
- IPSec SA

Both SAs are established between IPSec peers using the IKE protocol, but yet they each serve a separate purpose.

IKE SAs between peers are used for control traffic, such as negotiating algorithms to use to encrypt IKE traffic and authenticate peers. There is only one IKE SA between peers, and it usually has less traffic and a longer lifetime than IPSec SAs.

IPSec SAs are used for negotiating encryption algorithms to apply for IP traffic between the peers, based on policy definitions that define the type of traffic to be protected. Because they are unidirectional, at least two IPSec SAs are needed (one for inbound traffic and the other for outbound traffic). It is possible to have multiple pairs of IPSec SAs between peers to describe unique disjoint sets of IP hosts or IP data traffic. IPSec SAs also usually have more traffic and a shorter lifetime than IKE SAs.

IKE operates in two phases to establish these IKE and IPSec SAs:
- Phase 1 provides mutual authentication of the IKE peers and establishment of the session key
- Phase 2 provides for the negotiation and establishment of the IPSec SAs using ESP or AH to protect IP data traffic

ISAKMP and IKE are often used interchangably but they are not exactly the same. ISAKMP defines IPSec peer communication, message exchange between peers, and the state transitions they go through to establish their connection. IKE on the other hand defines *how* the key exchange is done.

##### IKE Phase 1 Operation
IKE Phase 1 offers two modes, Main and Aggressive. There are various parameters that are negotiated between the two peers, as defined by the IKE SA. The mandatory parameters are:
- encryption algorithm (DES, 3DES, AES)
- hash algorithm (HMAC version of the negotiated hash algorithm)
- authentication method (pre-shared key, digital signature, public key encryption with 4 encryptions, publick key encryption with 2 encryptions)
- Diffie-Hellman Group (specified by the group number)

Here is an example config on a Cisco router:
```
crypto isakmp policy 1
  encryption 3des
  hash md5
  group2
  lifetime
  authentication pre-shared
```
###### Main Mode
Main mode consists of 6 messages, 3 in each direction between the initiator and the responder.
- First exchange: The algorithms and hashes used to secure the IKE communications are agreed upon in matching ISAKMP SAs in each peer.
- Second exchange: Uses a Diffie-Hellman exchange to generate shared secret keying material used to generate shared secret keys and to pass nonces numbers sent to the other party and then signed and returned to prove their identity.
- Third exchange: Verifies the other side’s identity (the identity value is an IP address, an FQDN, an email address, a DNS or a KEY ID form in encrypted form).

The main outcome of main mode is matching ISAKMP SAs between peers to provide a protected pipe for subsequent protected ISAKMP exchanges between the IKE peers. The ISAKMP SA specifies values for the IKE exchange: the authentication method used, the encryption and hash algorithms, the Diffie-Hellman group used, the lifetime of the ISAKMP SA in seconds or kilobytes, and the shared secret key values for the encryption algorithms. The ISAKMP SA in each peer is bi-directional.

###### Aggressive Mode
Aggressive Mode has fewer exchanges with fewer packets than Main mode - it has a total of 3 exchanges. On the first exchange, almost everything is squeezed into the proposed ISAKMP SA values: the Diffie-Hellman public key (a nonce that the other party signs) and an identity packet, which can be used to verify identity via a third party. The receiver sends everything back that is needed to complete the exchange. The only thing left is for the initiator to confirm the exchange. The weakness of using the aggressive mode is that both sides have exchanged information before there’s a secure channel. Therefore, it’s possible to sniff the wire and discover who formed the new SA. However, it is faster than main mode.

###### Authentication Methods
As mentioned earlier, one of the parameters with the most impact on the IKE phase 1 exchange itself is the authentication method. The two most commonly used authentication methods are Pre-shared Key and Digital Signature.

####### Pre-shared Key Authentication
In this method, both the initiator and responder must have the same pre-shared keys; otherwise, the IKE tunnel will not be built due to the pre-shared key mismatch. The keys are agreed upon typically using an out-of-band technique.

One disadvantage of using pre-shared keys in IKE Main Mode is that, because the ID payloads are encrypted, the responder is not aware of the identity of the initiator. In remote access scenarios (such as telecommuting), the source IP address of the initiator is not known in advance to the responder. In such cases, Aggressive Mode is the *only* choice with pre-shared key authentication. Here is a sample configuration.
```
crypto isakmp policy 1
  encryption 3des
  hash md5
  group2
  lifetime
  authentication pre-shared
  crypto isakmp key ciscovpn 9.1.1.35
  crypto isakmp key wildvpn address 0.0.0.0 0.0.0.0
```
Due to its simplicity, this authentication method is widely deployed for mass IPSec VPN deployment.

####### Digital Signature Authentication
In the case of digital signatures, peers can be authenticated with public key signatures, either DSS or RSA. Currently, Cisco IOS only supports RSA. Public keys are usually obtained using certificates. IKE allows for the exchange of certificates between the initiator and responder.


##### IKE Phase 2 Operation
IKE Phas 1 creates the IKE/ISAKMP SAs while Phase 2 established the IPSec SA in each direction. Phase 2 is also reffered to as Quick Mode. At the completion of Quick Mode, the two peers should be ready to pass traffic using ESP or AH modes. Because an IPSecSA is unidirectional, there will be a minimum of two IPSec SAs between two IPSec peers.

###### Quick Mode
Quick Mode has three exchanges. All of these messages are protected by IKE, which means that all of the packets are encrypted and authenticated by the smae keys derived in IKE Phase 1.

1. The peer will send an IPSec Proposal this time which will include agreed upon algorithms for encryption, integrity, and what traffic is to be secured or encrypted (Proxy ID). The traffic that is to be secured will typically be defined as part of an ACL. Configuring the IPSec proposal is completely different than configuring the ISAKMP policy. the algorithms you use here can be completely different than phase 1. This could also be configured with a tunnel rather than an ACL.
2. This is the reply from the responder to the initiator which will include the chosen proposal and the Proxy IDs. The Proxy ID will probably be the inverse of the first Proxy ID that was sent.
3. This is a final acknowledgement. After this, there should be two unidirectional secure channels from one peer to the other peer

One thing to note is that there is a timeout for the IPSec SA where the encryption key is changed. The default for this is 1 hour. Another thing you can enable with IPSec is Perfect Forward Secrecy (PFS) which will help frequent changes of the encryption keys and if enabled, it can help cycle the keys out more regularly but when this is enabled, you might see more issues between different vendors. If the PFS group is configured in the IPSec configuration, it should match on both side and it should not be the same as the Diffie-Hellman group.

##### IPSec Packet Processing
Processing of packets on a router is typically just an implementation issue. RFC 2401 describes a general model that describes two databases that all IPSec implementations will use:
- Security Policy Database (SPD) - holds policy definitions that determines how traffic is treated taht passes between the peers
- Security Association Database (SADB) - contains the parameters associated with each active security association




