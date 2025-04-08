---
title: "Intuiting TLS"
---

How do you speak privately in a public setting? Think about it.

The internet's age-old problem; Anyone between you and the websites you visit can see (and even _edit_) all back-and-forth communication. Can you prevent the router at starbucks from seeing your login password? This is what stuff like Transport Layer Security (TLS, formerly SSL) attempts to solve.

But going back a bit: how _would_ you do it?

### Encryption

Well, the simplest answer is to speak in a code (cipher) known only to you and the website, where one side scrambles the message (encryption) and the other unscrambles it (decryption). A good start, but it's problematic; First, the cipher has to be relatively complex or people will figure it out (_"use one letter over"_ becomes painfully obvious after a while). Second, every website can't have the same cipher.. It would no longer be secret.

OK, new plan; Generate the cipher _randomly_ and _uniquely_ for each website. We have a [Random Number Generator](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator) (RNG) construct that can be created with a starting point (seed). Each website has its own seed. Encryption takes bytes from RNG and combines it with our message: `M + RNG(seed) = Enc`. Decryption does the inverse: `Enc - RNG(seed) = M`. The seed can be called a _"key"_ as it effectively locks & unlocks messages. Great!

Wait... How do you know the website's key? If it's sent over the internet, other's would see it too & create the same `RNG(key)` to decrypt our messages. Maybe it could be distributed before-hand? But then what about websites we visit for the first time? Quite the head-scratcher.

### Key Exchange

Fortunately, there's a neat algorithm to solve this: [Diffie-Hellman Key Exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange). It relies on an operation (op) that is both _"one-way"_ (`A op B = C` is easy, but the inverse `A = B ?? C` is very hard) and _"commutative"_ (`A op B = C` or `B op A = C`, order doesn't matter). Here's how it works:

The website and I generate our own random "private" key that's never shared (`K1` and `K2` respectively). We each compute our own "public" key using a known constant (`K1 op C = P1`) and share it with each other over the internet. Finally, we combine our own "private" key with the _other's_ "public" key (`K1 op P2 = S`) to get a "shared secret" key that's known only to the both of us.

Seems like magic, but look at what's happening: One side computes `K1 op (K2 op C)` while the other computes `K2 op (K1 op C)`. Since op is commutative, we arrive at the same result. Since op is one-way, the public keys `K op C` sent over can't realistically be reversed to find `K`.

Plug the "shared secret" key into an RNG and voila! We now have a way to setup a secret channel with any website! ... Or so you'd think. 

All of this only prevents someone from reading our messages. But what if they can edit them too?

### Authentication

There's two methods of compromise here:

1. When sending over the public keys, someone could swap our view of "the other's public key" to their own, resulting in us "secretely" talking to the middle-man instead of to each other. How do I trust that the other side of `website.com` is really them? (_Key Authenticity_)

2. When decrypting messages, someone could just replace the encrypted text with some that decrypts into an undesired message/response. Imagine my bank sends me an encrypted message asking to move funds. Someone making my response decrypt to "yes" instead of "no" is pretty bad. How do I trust the other side is saying what they mean, even with a valid shared secret key? (_Data Authenticity_)

I'll explain the solution for 2. then 1. as the latter is a bit more involved.

#### Data Authenticity

First, we'll need to introduce a hash function `H(x) -> y`. This is an operation that takes an input `x` and produces an output `y` that's resistant to _"pre-images"_ (finding `x` from only `y` is very hard) and _"second pre-images"_ (finding some `z` that also hashes to `y` is very hard).

Then, we hash the encrypted message along with the secret RNG key `H(Enc ++ key)`, sending both `Enc` and the result together. The hash result acts as a way the decryption side can use to double-check the message is right before accepting.

If we only hashed the encrypted message, the middle-man could also just hash their own malicious encrypted message. However, this hash includes the RNG key that's kept secret from the middle-man. Meaning they can't realistically compute a hash for their `MaliciousEnc` that satifies `H(MaliciousEnc ++ key)` to pass decryption.

Such as scheme is called **A**uthenticated **E**ncryption (optionally hashing some **A**dditional **D**ata). Or **AEAD** for short.

#### Key Authenticity

Alright, but how do we protect the shared secret key from being compromised?

Honestly? Not much you can do besides agree to something before-hand. A bit dissapointing that there's no clever solution here, but think about it; You can't know you're talking to someone if you don't already know who they are..

So then the straight-forward approach is **P**re-**S**hared-**K**eys (**PSK**) that both sides would either put directly into the AEAD (skipping Key Exchange), or use to randomize the Diffie-Hellman shared secret result. Problem here is that, again, it requires a PSK per website which is impractical.

### Certificates

Fortunately we can solve this one using two new primitives: `Sign(K, M) -> S` takes a private key and a message, producing a signature. This can then be fed into `Verify(P, M, S) -> bool` taking the public key and same message, returning if the signature really came from whatever private key was used to create the public key. How this works can be [gnarly](https://en.wikipedia.org/wiki/Digital_Signature_Algorithm#Operation), but just know that it builds upon similar concepts as the `op` in Diffie-Hellman.

Now, we introduce a trusted third-party (Root) who keeps track of who's who & has their own public/private key pair. Both sides have Root's public key pre-shared. I, as a client, only trust stuff `Sign`'d by Root. So the server gets their public key + website name signed by Root and passes that signature along when doing Key Exchange.

A middle-man changing the signature would fail my `Verify`. To properly intervene, they would have to get Root to sign _their_ malicious public key + the same website name. But Root already has the website registered to someone else and obviously won't let them. As long as I trust Root's signature, I can know whatever website im talking to is really them.

These "public key + website name" signatures can be called **Certificates** and they form a centralized trust system known as **P**ublic **K**ey **I**nfrastructure (**PKI**). You can also create a "chain of trust" by saying _"I trust anything signed by Root2 since they were signed by Root"_.

Certificates may also include things like "how long the public key is valid for" in the signature to minimize the off-chance of someone managing to brute-force guess the private key. It can be why web admins say things like _"my cert expired; need to renew it"_.

----

So that's what it takes, at a high level, to talk privately on the internet. It's a pretty hostile environment and TLS addresses this while also handling a bunch of edge cases I didn't cover like [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) and various [data/timing attacks](https://en.wikipedia.org/wiki/Transport_Layer_Security#Security).

If you're curious how it all works, I'd honestly recommend to start out by skimming tools like [LibHydrogen](https://libhydrogen.org/) and [Monocypher](https://monocypher.org/). They provide the implementations for the low level bits such as Key Exchange, AEAD, and Signing. After that, check out Julia Evans' [Toy version of TLS 1.3](https://jvns.ca/blog/2022/03/23/a-toy-version-of-tls/) or xargs' [The Illustrated TLS 1.3 Connection](https://tls13.xargs.org/) for how the protocol stiches it all together. There doesn't seem to be many learning resources for TLS (mostly specs and marketing). Let me know if there's any others.
