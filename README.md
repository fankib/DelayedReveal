# Delayed Reveal
Reveal a secret in the future.

# Introduction
This project aims to reveal a secret at a future point in time without further
actions by the owner of the data. To achieve this we relay on trusted
third parties where a predefined amount has to remain trustworthy. However, the power
of the third parties is limited to deny the decryption. Even if all parties are
compromised they can not decrypt the cipher. 

This readme provides an overview of the protocol and tries to answer all open questions
with the formal specification. The only thing left out is the implementation.

## Time in the future
The concept of time is manmade and may be change in future. Perfect future decryption
may relay on physical circumstances (cpu cycles, travel through space, ...) and is out of
scope for this project.

## Vision
Reveal encrypted data automatically in the future under the assumption that t out of n parties behave correctly
without any further interaction and still maintaining some usability in case the owner changes his intention.

## Overview of the protocol
The owner is the person who has some data to share in future. The naive approach is to upload
the data to a server and trust that the server will retain the data until the given future moment.
Then the trust lies on this only server. The target of this project is to enhance this naive
aproach with the share of trust among multiple parties. We split the secret into two parts and share
one part among the authorities. The other part is shared among the intended circle of recipients.

### Limitations
- The owner of the data may reveal the data at any point
- Are (n-t) or more parties compromised they may deny decryption.
- Are t or more parties compromised they may reconstruct their part of the secret.

### Features
The first limitation indicates that the owner has some control over the data. We want to extend the protocol
to achieve more usability introducing control over the data under encryption and the time of reveal.

The owner should be able to provide new versions of the data in case something has to be updated.
Further we want that the latest version will be decrypted only.
In addition the owner should be able to exchange the time oracle with any other moment in the future
or the past.

_Note: This part of the protocol may introduce the needed usabiltity but it makes it also vulnerable to replay attacks
which have to be addressed later._

### Assumptions
- The network layer is assumed to be confidential.
- The platform of the owner is assumed to run the correct code.

### Building blocks
For the encryption of the data we use AES. The symmetric key will be derived of
two xored secrets: the first one will be shared among the authorities using shamirs secret sharing.
The second one is a random bit array shared among the circle of recipients.
To reauthenticate the owner for manipulating an already shared file we work with digital signatures. 

To summarize:
- AES
- Shamir's Secret Sharing
- Schnorr Signatures (or something else)
- SHA-256

### Organization 
One remaining question is how the authorities are organized in order to work together. In the first version of
the protocol we assume a fixed amount of authorities where each one knows each other. N is fixed and
known to anybody.

_Note: If an authority joins later he will fit into the protocol like a misbehaving authority which does not
reveal his secret (as he has none to contribute) and t secrets will be found when there are n+1 authorities._

# Formal Specification
## Parties
- Owner: O
- Authorities: A1, A2, An
- Number of Authorities: n
- Number of required trustworthy Authorities: t

## Payloads 
- Data: D
- Encrypted Data = Cipher: C
- Definition of the future time oracle: F
- Private Signature Key: SK
- Public Signature Key: PK
- Secret = Shared Secret: S
- Shared Secret with Authroity A1: S1
- Shared Secret with Authority A2: S2
- Shared Secret with Authortty An: Sn
- Offline Secret for circle of recipients: SO

## Protocol for 128 bit security

### Preparation phase
- The owner O creates a Secret S and an offline Secret SO:
	- S <- rnd(2^256)
	- SO <- rnd(2^128)

- The owner O derives an AES-128 Key out of S and SO and creates the cipher C:
	- AES-Key <- leastSignificantBits_128(S) xor SO
	- C <- aes_128(AES-Key, D)

- The owner O defines a time oracle F.
	- example: {"type":"not-before", "value":"2019"}
	- example: {"type":"not-before-btc-block", "value":"25252525"} 

- The owner O creates a PK for a suited signature scheme.
	- The signature scheme has to be non-malleable as we have to handle
	  replay attacks.
	- I suggest a suitable prime order subgroup of the later used field for
	  the secret sharing. Maybe hash-then-sign using Schnorr Signatures?

- The owner O persists the SK and SO for further usage.

### Sharing phase
- The owner O uses Shamir's secret sharing in the role of the dealer and thus:
	- defines a polynom p(x) = c0 + c1*x + c2*x^2 + ... + ct-1*x^(t-1) mod M in a finite field of prime order which is suitable to contain the Secret S.
	  The polynom has degree t-1 defined with t coefficients (where p(0)=c0=S is the secret and t-1 coefficients are choosen randomly)
	- calculates the shared secrets S1 to Sn by evaluating the points S1=(1, p(1)) to Sn=(n, p(n)) on the curve.

- The owner signs (C, F, S1, PK) for Authority A1 and shares this tuple confidentially with Authority A1
- The owner signs (C, F, Sn, PK) for Authority An and shares this tuple confidentially with Authority An

- The owner O generates a Link or QR code to share with his circle.
	- The link may be in the format http://delayed-decryption.com/get/_PK_#_SO_.
	  Where the software can retrieve C from an arbitrary Authority for the given PK.
	  As the offline secret is part of the hashtag it will not be sent
	  to the server.


### Update the data
- The owner O decides to exchange the Data D with Data D2: he creates a new Secret S\_2 and creates Cipher C\_2 (using the old SO as the circle has not changed).
- The owner O calculates the new shared secrets S1\_2 to Sn\_2.
- The owner O signs (C\_2, S1\_2) and shares this tuple confidentially with Authority A1.
- The owner O signs (C\_2, Sn\_2) and shares this tuple confidentially with Authority An.
- Each authority Ai checks the signature and replaces their copy of the Cipher C with C\_2 and their Shared Secret Si with Si\_2.

### Update the time oracle
- The owner O decides to exchange the time oracle F with another definition F2.
- The owner O signs (F2) and share it among the Authorities A1 to An.
- Each authority Ai checks the signature and replaces their copy of the Oracle F with F2

### Replay attacks
The update of data and the update of the time oracle enable replay attacks when the signature is valid
but the payload isn't anymore.

_Note: the authorities have to guarantee that they do never replace an Oracle F2 with F as this would
open a replay attack where an attacker could send a valid signature for a, now invalid, oracle F (any corrupted authority
will be in posession of valid signatures for F)._
_Depending on the used signature scheme there may be multiple approaches to handle replay attacks._

### Reveal phase
- When the time oracle F evaluates to true, the authorities A1 to An will reveal their secret upon request. If t Authorities
  have reveald their secret, anybody will be able to reconstruct S and derive the AES Key when in possession of the Offline Secret SO.
  Having the AES-Key the recipient is now able to decrypt C and retrieve Data D.

# Implementation
The algorithms may be implemented in JavaScript to run in the browser and provide this service
to anybody without the need of any software installatioan. HTML and the Web still have no PKI nor
any guarantee in software integrity so code injections are possible when using the software of a corrupted provider. 
This issue may be solved in future with a trust-on-first-use JS-Approach or dedicated software for each 
supported platform.


