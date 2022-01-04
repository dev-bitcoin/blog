---
title: "Schnorr basics"
subtitle: "Explaining Schnorr signatures with simplified maths"
date: "2022-01-03"
authors:
  - name: Kalle Rosenbaum
    github: kallerosenbaum
    twitter: kallerosenbaum
tags:
  - "Schnorr"
  - "signature"
  - "BIP340"
mathjax: true
images:
    - "/post-data/schnorr-basics/sig-overview.png"
appearedfirston:
  label: "popeller.io"
  url: "https://popeller.io/schnorr-basics"
---

If you're having a difficult time wrapping your head around Schnorr
signatures, you're not alone. In this post, I attempt to explain
Schnorr signatures at a level that I myself appreciate, and
hopefully, you'll find it valuable too.

<!--more-->

## What's Schnorr?

Schnorr is a new signature scheme in Bitcoin that got activated in the
taproot upgrade. Schnorr has some nice properties listed in
[BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#motivation), but I won't reiterate those here.

## Signing and verifying

A signature scheme consists of two actions, signing and verification
as the following diagram shows.

{{< center-figure
    src="/post-data/schnorr-basics/sig-overview.svg"
    caption="Left side: signing using a message and a private key. Right side: verification that the message (the cat) was signed with the correct private key."
    height=80%
    width=80% >}}

The signer has a private key and a message to sign (for example, a
Bitcoin transaction or a cat picture) and produces a signature. The
verifier has the public key corresponding to the private key that the
signer used, the message, and the signature. The verification process
makes sure that the signature was created with the correct private
key without knowing the private key.

If you want a more in-depth explanation, please visit [the second half of chapter 2 of Grokking Bitcoin](https://rosenbaum.se/book/grokking-bitcoin-2.html#_digital_signatures).

### Signer

We're now going to see how a Schnorr signature is created. It
represents the left side of the diagram above. I assume that you're
familiar with how a key pair is created using a random number
generator and an elliptic curve. If not, I suggest you read
[section 4.8 of Grokking Bitcoin](https://rosenbaum.se/book/grokking-bitcoin-4.html#public-key-math).

Suppose that you want to make a signature for the cat picture. The
picture is the message, $m$, to sign. Your private key is $p$ which belongs to your public key $P$.

The first thing you'll do is to draw a random number, $r$, that we'll
call the nonce (for "Number Once"). Then you'll treat $r$ as if it was
a private key, which means that you can generate the corresponding
public key by multiplying it with $G$, which is the _generator
point_. At this stage, you have the following:

$$
\begin{align}
p & : \text{private key} \\\\
P = pG & : \text{public key} \\\\
m & : \text{message} \\\\
r & : \text{random nonce} \\\\
R = rG & : \text{nonce commitment}
\end{align}
$$

$R$ is your _nonce commitment_ which will become the first part of
the final signature, and $r$ must remain secret
([explained later](#why-r-secret)) and never be reused (also
[explained later](#nonce-reuse)). You've prepared everything you need
to make the signature. You'll do it in two steps. Step 1 is to
calculate a so-called _challenge hash_, $e$:

$$
e = H(R || P || m)
$$

The challenge hash is the hash of the challenge, which is the
concatenation of $R$, $P$, and $m$. These components will all be
available to the verifier. Using the challenge hash, you can now do step 2:
Calculate the _challenge response_, or simply _response_, $s$, which
is the second part of the signature (Thanks to Anthony Towns,
Ruben Somsen, and nothingmuch on Twitter for
[their help with the name _response_](https://twitter.com/kallerosenbaum/status/1472515231050080266).):

$$
s = r + ep \tag{1} \label{respeqn}
$$

Finally, your signature is

$$
(R,s)
$$

You send the cat picture, $m$, and your signature $(R,s)$ to your
friend, Fred.

### Verifier

Fred wants to make sure the cat picture hasn't been compromised during
transfer and that it really originates from you, the only one with
access to your private key $p$. He has access to $P$, $m$, $R$, and
$s$. Of course, he also has access to $G$ because that's a widely
known constant. From this information, he can calculate the challenge
hash and verify that the _verification equation_ balances:

$$
\begin{align}
e & = H(R || P || m) \\\\
sG & = R + eP \tag{2} \label{vereqn}
\end{align}
$$

**If this equation balances, Fred can be sure that the signature was
made with $p$**. Note that the verification equation, equation
$(\ref{vereqn})$, is the response equation, $(\ref{respeqn})$, where
both sides are multiplied by $G$. Starting with the response equation,
we get

$$
\begin{align}
&s = r + ep \iff sG = (r+ep)G \\\\
&\iff sG = rG + epG \iff sG = R + eP
\end{align}
$$

As you can see: If the response equation holds, the verification
equation holds. Likewise, if the verification equation holds, the
response equation also holds. Thus, when Fred verifies the equation
based on points on the elliptic curve, the verification equation, he
also implicitly verifies that the response equation, based on scalars,
holds.

When Fred has verified the signature, he can enjoy the cat picture,
fully confident that it's actually the same picture as you sent him.

## Why is $r$ secret? <a id="why-r-secret"></a>

You might wonder why the nonce $r$ must be kept secret. You might even
wonder why it's needed at all? Let's start with the latter. Let's remove $r$ from the process and see what happens.

You would create the signature as follows:

$$
\begin{array}{}
e = H(P || m) \\\\
s = ep
\end{array}
$$

The signature would consist only of $s$. Fred would then verify your signature as:

$$
\begin{array}{}
e = H(P || m) \\\\
sG = eP
\end{array}
$$

That equation holds, but it would also allow Fred to extract the
private key $p$ since he knows both $s$ and $e$. He'll take the
response equation and solve it for p:

$$
s = ep \iff p = \frac{s}{e}
$$

OK, we need the nonce to prevent Fred from figuring out your private
key, but why must we keep the nonce secret? Why the hassle of using
the nonce commitment, instead of the nonce itself?

It's for the same reason. Suppose that that the nonce, $r$, was made
available to Fred, then he could figure out $p$ by:

$$
\begin{array}{}
e = H(R || P || m) \\\\
s = r + ep \iff p = \frac{s-r}{e}
\end{array}
$$

So Fred could calculate $p$ by subtracting $r$ from $s$ and dividing
the result by $e$.

By revealing just the nonce commitment, $R$, to Fred, we make sure
that Fred can't calculate $p$ while at the same time allowing him to
verify that $p$ was used to generate the signature.

## Don't reuse nonces <a id="nonce-reuse"/>

Even if you keep the nonce secret, you may still leak your private key
if you use the nonce twice for the same private key. Suppose that you
make two signatures with the same nonce and private key as follows:

$$
\begin{array}{lr}
e = H(R || P || m) & e' = H(R || P || m')\\\\
s = r + ep & s' = r + e'p
\end{array}
$$

Then you give the signatures $(R,s)$ and $(R,s')$ to the verifier. The
verifier can then use simple arithmetic to calculate your private
key. He can set up an equation system with two equations and two
unknown as follows:

$$
\begin{align}
    s &= r + ep\\\\
	s' &= r + e'p
\end{align}
$$

This is solvable for $p$ by subtracting $s'$ from $s$:

$$
\begin{align}
&s-s'=r+ep-r-e'p \\\\
&=(e-e')p \implies p=\frac{s-s'}{e-e'}
\end{align}
$$

As you can see, the verifier will be able to extract the private
key. Lesson learned: Don't reuse nonces.

If the nonce is reused, but for different private keys, $p$ and $p'$,
the above equation system wouldn't be solvable because you'd have
three unknowns, $p$, $p'$, and $r$, but only _two_ equations.

## What's with the challenge? <a id="challenge"/>

The challenge hash, $e$, is the hash of the challenge $R||P||m$. Why
do we use this particular challenge? Let's look at the three components separately.

### $m$

The message to sign is $m$, so it's really important that $m$ is
somehow committed to by the signature. If we'd remove $m$ from the
challenge, the "signature" would be valid for any message.

### $R$

(Thanks to [waxwing](https://x0f.org/@waxwing/107491024866468566) and
[A J Towns](https://mastodon.social/@ajtowns/107491605500890702) for
their help in sorting this out.)

To make sure that no one but the owner of the private key can create a
signature, the challenge must contain the nonce commitment
$R$. Suppose that the challenge didn't include the nonce commitment,
then a signature can be trivially forged by anyone with access to the
public key $P$. They can make up an arbitrary $s$ and do:

$$
\begin{align}
&e = H(P||m)\\\\
&s = \text{any number} \\\\
&sG=R+eP \implies R=sG-eP
\end{align}
$$

The last equation is the "verification equation", but solved for
$R$. The right side of that equation contains only known variables,
$s$, $e$, and $P$. Thus the signature $(R,s)$ is valid.

With $R$ in the challenge, it's impossible to solve the
verification equation for $R$ because $R$ is part of the challenge
$e$. It's hard to find an $R$ such that $R=sG-H(R||P||m)P$.

### $P$

Let's finally see what $P$ is doing in the challenge. Suppose that we
didn't have $P$ in the challenge, $e=H(R||m)$ and that the signature
$(R,s)$ is valid for public key $P$ and message $m$. Then the
signature $(R,s')=(R,s+ex)$, where $x$ is an arbitrary number, would
be valid for a public key $P'=P+xG$ and message $m$. Let's look at
why:

$$
\begin{align}
&e = H(R || m) \\\\
&s'G=R+eP' \iff (s+ex)G=R+e(P+xG) \iff \\\\
&sG+exG=R+eP+exG \iff sG=R+eP+exG-exG \iff \\\\
&sG=R+eP
\end{align}
$$

This is known as a related-key attack. If you're familiar with how
extended public key derivation in
[BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
works, you might see the potential danger. For a refresher, here's the
general idea:

{{< center-figure
    src="/post-data/schnorr-basics/xpub-derivation.svg"
    caption="Deriving a child extended public key from a parent extended public key, The orange paper strips are known as chain code, described in detail in Grokking Bitcoin. You just need to know that they're 256-bit numbers."
    height=80%
    width=80% >}}

This means that if an attacker knows the parent extended public key
(xpub), and a valid signature for a child key, then the attacker can
use this trick to forge signatures for the parent xpub, as well as any
child xpubs that can be derived from the parent xpub. This is a
problem not only for BIP32, but for many schemes using public key
addition somehow, for example, Taproot
([BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs)). For
a bit more details, please visit
[the Design section of BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#Design).

## Next steps

In the next post, I'll show how Schnorr signatures can be used in a
multisignature setting to produce a signature that looks just like a
normal single signature. This is very practical in Bitcoin because it
reduces resource requirements for verifying the blockchain.


_This is a cross-post from [my personal blog](https://popeller.io/schnorr-basics)._
