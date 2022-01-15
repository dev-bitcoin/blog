---

title: "Trustless Lightning Payments for Content"
subtitle: "How PTLC payment channels in Lightning enable decentralized social media."
date: 2022-01-07T00:00:00+00:00
authors:
  - name: Jonathan Zernik
    github: yzernik
    twitter: yzernik
tags:
  - "Lightning"
  - "PTLC"

mathjax: true
images:
  - "/post-data/trustless-lightning-payments/cover-image-2.png"

appearedfirston:
    label: "squeaknode.org"
    url: "https://squeaknode.org/lightning/ptlc/bitcoin/2022/01/07/trustless-lightning"

---

The existence of the Lightning Network enables new types of online communities
where participation and access to content is dependent on individual users making
Bitcoin micropayments. These new communities can operate without a central
authority, and without a middleman taking a cut of the profit.

In this post, I will briefly describe the weaknesses of the existing models
for Lightning monetization, and how these problems will be solved when PTLC
payment channels are the standard in Lightning Network.

#### How Lightning incentivizes content

If we imagine a world where social media is decentralized, content will no
longer be hosted by large companies like Twitter, Facebook, Spotify, and
Youtube. Instead, we will see individuals and independent hosting providers
relaying and aggregating content that is signed cryptographically by individual
users. This raises the question of how will these relays/aggregators be
incentivized to host content. Lightning payments are the obvious answer.

#### What is wrong with the tipping model?

There are currently attempts to create models of monetization that use
Lightning network to reward content creators:

- Value-For-Value (Used in Podcasting 2.0 apps like Fountain)
- Tipping (Used in Zion)

These apps use a voluntary payment model, where any user can access content
for free, and then decide how much they want to pay in tips to reward the
content creator.

This leads to a situation where the majority of users are consuming the
content for free, and a small generous minority of users is generating all
of the revenue for the content creator. The revenue of the content creator
is totally dependent on the generosity and altruism of their consumers.

It is the digital equivalent of "busking" on a sidewalk, and hoping that people
will drop a few coins in your hat.


#### Moving to a pay for content model

A better, fairer model would be one where the consumer is required to make a
payment to get access to the content.

A naive approach to accomplish this would be to simply encrypt the content,
and then create a Lightning invoice with the decryption key as the invoice
preimage (using HTLCs), and share the invoice with the consumer. Then, when the consumer
pays the invoice, they would obtain the decryption key, and use it to unlock
the content.

<!-- In the current implementation of Lightning in LND (using HTLCs), the preimage -->
<!-- would be specified in the `r_preimage` field of the `AddInvoice` RPC method. -->

{{< center-figure src="/post-data/trustless-lightning-payments/htlc-example-4.png" height=600 width=600 >}}

Unfortunately, this approach requires the consumer to trust the honesty of the
content seller. A malicious seller could create an invoice using an invalid
decryption key as the preimage. The buyer would make the Lightning payment, and
only realize after the payment that they got an invalid decryption key. The
buyer has no way to verify the validity of the invoice until it is too late,
and they have already lost their funds.

This model of monetization can only work if the buyer is able to trust the seller.
This might be true if the seller is a large, reputable institution, but this
model breaks down if we try to extend it to a peer-to-peer decentralized
network, where all connections are assumed to be trustless.

#### Making payments trustless with PTLCs

Fortunately, there is a way to make the payment trustless for both the buyer
and the seller, but it requires that Lightning use PTLC payment channels.

With PTLCs, the seller specifies a 32-byte scalar value ($s$) as the "preimage"
at the creation-time of the invoice. When the buyer decodes the invoice payment
string, they will obtain the payment point ($p = s*G$). The buyer will only
obtain the "preimage" $s$ after the payment is settled.

So now that we know that PTLC invoices use elliptic curve points, we can
take advantage of the distributive property of scalar multiplication
on elliptic curve points:

$$ (s1 + s2)\*G = s1\*G + s2\*G $$

where $s1$ and $s2$ are scalars, and $G$ is an elliptic curve generator point.

The basic idea for selling content is as follows:

* Alice has a piece of content she wants to be sellable.
* Alice generates a scalar value $s1$ to use as a symmetric encryption/decryption key, and encrypts the content.
* Alice calculates the point $p1$ on an elliptic curve $G$ by calculating $p1 = s1*G$.
* Alice makes $p1$ publicly available for anyone interested in buying the content.
* Alice shares the encrypted content with a relay named Carol.
* Bob downloads the encrypted content from Carol, and requests an invoice to unlock the content.
* Carol generates a new scalar value $s2$, and creates a *PTLC* Lightning invoice with $s1 + s2$ as the preimage.
* Carol sends Bob $s2$ and the Lightning invoice as a payment request string.
* Bob decodes the payment request string to get the payment point of the invoice, call it $p3$.
* Bob calculates $s2*G$. If it is equal to $p3 - p1$, then Bob knows that the invoice is valid.
* Bob pays the Lightning invoice, and gets the value of $s1 + s2$ as the preimage.
* Bob calculates $s1 = (s1 + s2) - s2$ to get the decryption key, and decrypts the content.

If Carol tries to give Bob an invalid invoice, Bob will know before he pays the invoice.

The above example assumes that Bob is able to trust that the value of $p1$ is valid.

{{< center-figure src="/post-data/trustless-lightning-payments/ptlc-example-3.png" height=600 width=600 >}}

Now any node in the network can relay content from any creator, and earn a profit by
selling it trustlessly to any consumer. The consumer does not have to trust that the
relay is an honest actor, and vice versa. The only requirement is that the payment
point generated by the original content creator must be publicly known and trusted.


#### Payment for content with Squeak Protocol

The [Squeak Protocol](https://github.com/squeaknode/squeak/blob/master/docs/PROTOCOL.md)
is a peer-to-peer protocol that extends the idea described in the previous section
for sharing sellable, encrypted, signed posts. The result is a trustless,
decentralized alternative to Twitter with Lightning monetization.

With the Squeak Protocol, any user can download an encrypted post (called a "squeak"),
and verify several properties of the squeak:

1) The squeak has a unique hash that is the same across all nodes
2) The squeak was created and signed by a certain person, and the original content has not been altered.
3) The squeak was created after a certain time (based on the block hash).
4) The squeak was created in reply to another squeak (or not).
5) The squeak was encrypted using a scalar that corresponds to a certain
elliptic curve point. Any lightning invoice that matches the payment point
is guaranteed to decrypt the content.

After these properties have been verified, the consumer can request an invoice from
the seller that will unlock the decrypted content of the squeak (after payment):

{{< center-figure src="/post-data/trustless-lightning-payments/locked-squeak.png" height=600 width=600 >}}

If the consumer then clicks the "buy" button, they will be presented with an offer
that they received from the seller:

{{< center-figure src="/post-data/trustless-lightning-payments/received-offer.png" height=600 width=600 >}}

Then, if the consumer decides that the squeak is worth the price, they can pay the
invoice to unlock the squeak:

{{< center-figure src="/post-data/trustless-lightning-payments/unlocked-squeak.png" height=600 width=600 >}}

The consumer now has a copy of the unlocked squeak on their own node, and
they can now re-sell the squeak to other nodes in the network.

#### How squeaks work under the hood

There are two components in a squeak:

* The squeak header
* The external fields outside the header.

A squeak header has the following fields:

Field Size | Description | Data type | Comments
--- | --- | --- | ---
4 | nVersion | int32_t | Squeak version information
32 | hashEncContent | char[32] | The hash value of the encrypted content of the squeak
32 | hashReplySqk | char[32] | The hash value of the previous squeak in the conversation thread or null bytes if squeak is not a reply
32 | hashBlock | char[32] | The hash value of the latest block in the blockchain
4 | nBlockHeight | int32_t | The height of the latest block in the blockchain
33 | pubKey | char[33] | Contains the public key of the author
33 | recipientPubKey | char[33] | Contains the public key of the recipient if squeak is a private message
33 | paymentPoint | char[33] | The payment point of the squeak derived from the decryption key on the secp256k1 curve.
16 | iv | char[16] | Random bytes used for the initialization vector
4 | nTime | uint32_t | A timestamp recording when this squeak was created
4 | nNonce | uint32_t | The nonce used to generate this squeak

There are also external fields outside the header:

Field Size | Description | Data type | Comments
--- | --- | --- | ---
1136 | encContent | char[1136] | Encrypted content
64 | sig | char[64] | Signature over the squeak header by the author

There is also a squeak hash, which is derived from the bytes of the squeak header using SHA256.

### How a squeak is created

When a user creates a squeak, the following happens:

* An encryption/decryption key is generated as a random scalar value.
* A random initialization vector is generated.
* The post content is encrypted with a symmetric-key algorithm using the encryption key and the initialization vector.
* A hash is calculated over the encrypted ciphertext.
* The payment point is calculated from the encryption key scalar value on an elliptic curve.
* A new nonce is generated.
* The public key of the author is derived from their private key.

Also,

* The latest block height and block hash are fetched from the Bitcoin blockchain.
* The hash of another squeak (to make a reply) can also be provided by the author.

All of these values are used to populate the squeak header. After the header is created:

* The squeak hash is calculated from the header.
* The private key of the author is used to sign the squeak hash.
* The signature is then attached to the squeak.

The [MakeSqueak](https://github.com/squeaknode/squeak/blob/fde192b8b59b59090c6d2eb39f87d6b41b15c05e/squeak/core/__init__.py#L415-L462) function looks like this:

```python
def MakeSqueak(
        private_key: SqueakPrivateKey,
        content: bytes,
        block_height: int,
        block_hash: bytes,
        timestamp: int,
        reply_to: Optional[bytes] = None,
        recipient: Optional[SqueakPublicKey] = None,
):
    """Create a new squeak.
    Returns a tuple of (squeak, secret_key)
    private_key (SqueakPrivatekey)
    content (bytes)
    block_height (int)
    block_hash (bytes)
    timestamp (int)
    reply_to (Optional[bytes])
    recipient (Optional[SqueakPublickey])
    """
    secret_key = generate_secret_key()
    data_key = sha256(secret_key)
    if recipient:
        shared_key = private_key.get_shared_key(recipient)
        data_key = xor_bytes(data_key, shared_key)
    initialization_vector = generate_initialization_vector()
    enc_content = EncryptContent(data_key, initialization_vector, content)
    hash_enc_content = HashEncryptedContent(enc_content)
    payment_point_encoded = payment_point_bytes_from_scalar_bytes(secret_key)
    nonce = generate_nonce()
    author_public_key = private_key.get_public_key()
    squeak = CSqueak(
        hashEncContent=hash_enc_content,
        hashReplySqk=reply_to or b'\x00'*HASH_LENGTH,
        hashBlock=block_hash,
        nBlockHeight=block_height,
        pubKey=author_public_key.to_bytes(),
        recipientPubKey=recipient.to_bytes() if recipient else b'\x00'*PUB_KEY_LENGTH,
        paymentPoint=payment_point_encoded,
        iv=initialization_vector,
        nTime=timestamp,
        nNonce=nonce,
        encContent=enc_content,
    )
    sig = SignSqueak(private_key, squeak)
    squeak.SetSignature(sig)
    return squeak, secret_key
```

### Properties of a squeak

After a squeak is created, it can be shared and validated on any node.

A validated squeak has the following properies:

* The squeak hash is a unique identifier for the squeak, because it is derived from the immutable fields of the squeak header.
* The pubkey embedded in the squeak belongs to the author of the squeak (proved by the signature).
* None of the fields in the header were modified after being signed by the author (that would change the hash, and the signature would become invalid if that happened).
* The encrypted content field was not modified after being signed by the author (that would also result in the signature becoming invalid).
* The Bitcoin block hash and block height (if valid) prove that the squeak was created after that block was mined.

### How a user interacts with a squeak

For example, if Alice authored a squeak, and Bob obtained a copy of the squeak:

* Bob will know that the squeak was authored by Alice.
* Bob will know that the squeak was created after a certain time (given by the block hash).
* Bob will know if the squeak is a reply to another squeak.

However, because Bob does not have the decryption key:

* Bob will not be able to see the content of the squeak.

### How a user unlocks the content of a squeak

Now Bob has a choice to make. Is he interested in reading the squeak that Alice authored, given everything he knows about the squeak?

If Bob wants to unlock the content, he can send a message to all of his peers in the network expressing interest in buying the decryption key for the squeak.

Bob's peers in the network (only those who already have a copy of the decryption key) will respond to Bob by sending him invoices as described in the earlier section.

Bob can now browse through the offers, and make a payment to any peer that offered him a valid invoice. Bob knows which invoices are valid because he can validate against `paymentPoint` field of the squeak header, using the elliptic curve math described earlier.

When Bob makes a Lightning payment to one of the sellers, he will obtain the preimage of the invoice. This preimage can be used to get the decryption key:

* The preimage is $s1 + s2$, as described earlier.
* Bob already knows $s2$, because the seller sent it to him.
* Bob calculates $s1 = (s1 + s2) - s2$ to obtain the decryption key.
* Bob can then decrypt the content of the squeak.

Now Bob can read the content of the squeak!

The [GetDecryptedContent](https://github.com/squeaknode/squeak/blob/fde192b8b59b59090c6d2eb39f87d6b41b15c05e/squeak/core/__init__.py#L242-L261) method of the Squeak class looks like this:

```python
    def GetDecryptedContent(
            self,
            secret_key: bytes,
            authorPrivKey: Optional[SqueakPrivateKey] = None,
            recipientPrivKey: Optional[SqueakPrivateKey] = None,
    ):
        """Return the decrypted content."""
        CheckSqueakSecretKey(self, secret_key)
        data_key = sha256(secret_key)
        if self.is_private_message:
            if recipientPrivKey:
                shared_key = recipientPrivKey.get_shared_key(self.GetPubKey())
            elif authorPrivKey:
                shared_key = authorPrivKey.get_shared_key(self.GetRecipientPubKey())
            else:
                raise Exception("Author or Recipient private key required to get decrypted content of private squeak")
            data_key = xor_bytes(data_key, shared_key)
        iv = self.iv
        ciphertext = self.encContent
        return decrypt_content(data_key, iv, ciphertext)
```


#### Try it out for yourself

There is a working implementation of the core Squeak Protocol as a Python library here: https://github.com/squeaknode/squeak

And there is an implementation of a full node running the Squeak Protocol here: https://github.com/squeaknode/squeaknode

You can also run your own Squeaknode instance on [Umbrel](https://getumbrel.com/), but
it is not yet completely trustless because it still uses HTLCs.

[github repo]: https://github.com/squeaknode/squeak
[cross-post]: https://squeaknode.org/lightning/ptlc/bitcoin/2022/01/07/trustless-lightning
[website]: https://squeaknode.org/


