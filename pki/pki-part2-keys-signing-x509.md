# PKI With No Headache (Part 2): Asymmetric Keys, Digital Signing & X.509

## Asymmetric Keys

The foundation of an **X.509 certificate** is its use of **asymmetric keys**, where a **public key** is mathematically derived from a **private key**. This relationship is intentionally one-way — it is computationally infeasible to reverse the process and retrieve the private key from the public key.

In practice, the **private key** is used to create digital signatures, while the **public key** is used to verify them. This asymmetry enables secure communication, authentication, and trust without the need to share secrets.

## Digital Signing vs Encryption

At first glance, digital signing may seem similar to encryption, but it serves a completely different purpose. **Encryption** ensures **confidentiality** by hiding the contents of a message entirely. In contrast, **digital signing** focuses on **authenticity** and **integrity** — the message remains visible, but a separate piece of data, the **signature**, is attached.

To create this signature, a hash of the message is first computed (using algorithms like SHA or MD5), producing a unique fingerprint of the content. Then, only this hash is encrypted using the **private key**, resulting in the digital signature.

## Signing Uses Encryption

You might say, “Wait — didn’t we encrypt something using the private key when creating the signature? Doesn’t that make it encryption?”

Well, yes — technically, something *was* encrypted. But here’s the key point: it wasn’t the message itself that was encrypted, but rather a **hash** — a compact fingerprint of the message.

This distinction is important. In signing, encryption is used as a tool to prove authenticity, not to hide content.

## What a Signature Really Means

Digital signing is like saying to the other side:

*"Here’s the data. You compute the hash yourself — I’ve already done the same and encrypted my result with my private key. Now compare your result with mine."*

Since the signature was created using the **private key**, and only the corresponding **public key** can verify it, this proves that the signature came from the expected source and hasn’t been tampered with.

If the **hashes match**, the recipient can trust that the data is both **authentic** and **intact**.

## Why Symmetric Keys Still Matter

Another question you might ask:  

*"Why not simply encrypt the entire message with the private key, so the recipient can skip hashing and just decrypt it using the public key?"*

While that might sound reasonable, it doesn’t hold up in practice. **Asymmetric encryption is far slower and less efficient** than symmetric encryption — especially when handling large amounts of data.

That’s why **symmetric encryption** — where the same key is used for both encryption and decryption — remains the preferred method for securing actual message contents.

## How HTTPS Applies Everything

It’s **highly unlikely** that someone saved this blog and emailed it to you — you’re probably reading it on a **secure website**, and the URL most likely starts with **https**. That means you're using **TLS/SSL over HTTP**. In this setup:

- **Asymmetric encryption** is used first to securely exchange a symmetric key  
- Then, **symmetric encryption** takes over to efficiently handle the actual data transfer

**Authenticity** and **integrity** are ensured through digital certificates (X.509 v3) and asymmetric keys, while **confidentiality** is provided by the symmetric key.

Altogether, this means your connection is secure and trustworthy.

## From Keys to Certificates

Suppose you need an X.509 v3 digital certificate to secure a web server. Naturally, the first step is to generate an **asymmetric key pair**. The next step is just as important: you’ll need to send your **public key**, along with some identifying details, to a **Certificate Authority (CA)**.

But how? Email? Of course not — in the world of PKI, everything is governed by strict, standardized procedures.

This means you’ll need to follow a **well-defined format and protocol** to make that request. And that’s exactly where we’re headed next.

## Certificate Signing Request (CSR)

A **Certificate Signing Request (CSR)** packages your **public key** along with your **domain name**, **organization info**, and other extensions into a formal, signed request. 

Even in Part 1 of this series, the **Intermediate CA** had to create a CSR — which was then **signed by the Root CA**. The same rule applies to you.

Unless you're a **Root CA**, you can’t skip this step. Root CAs are the only entities that issue and sign their own certificates. That’s what makes a **Root certificate** truly **self-signed**.

## Certificate Signing with Intermediate CA

In Part 1, we created an **Intermediate CA** — a subordinate certificate authority trusted by the Root CA. Now it’s time to put it to work.

But before signing anything, we need to **generate a private key** for the web server. This key will be used to create a CSR, which the Intermediate CA will eventually sign.

### Step 1: Generate Private Key

```bash
openssl ecparam -name prime256v1 -genkey -noout -out web-server.key
```

This command generates an **ECC private key** using the **NIST P-256 curve** and saves it to `web-server.key`.

By default, OpenSSL generates RSA keys — but here, an **ECC key** is created **intentionally** for its **compact size**. This private key will later be used to generate a **Certificate Signing Request (CSR)**.

### Step 2: Create the CSR

```bash
openssl req -new -key web-server.key -out web-server.csr -subj "/CN=www.example.com"
```

This command creates a **Certificate Signing Request (CSR)**. It bundles your **public key** and **domain name** (specified in the **Common Name** field) and signs it using your **private key**.

The resulting file, `web-server.csr`, is what you'll send to a Certificate Authority (CA) for signing.

### Step 3: CA to Sign the CSR

```bash
openssl x509 -req -in web-server.csr \
  -CA inter.crt -CAkey inter.key -CAcreateserial \
  -outform DER -out web-server-der.crt -days 825 -sha256 \
  -extfile <(printf "keyUsage=digitalSignature,keyEncipherment\n\
extendedKeyUsage=serverAuth\n\
subjectAltName=DNS:www.example.com,DNS:example.com")
```

#### Command Breakdown

- `x509` — You're working with **X.509 certificates**
- `-req` — Input is a **CSR** that needs to be **signed**
- `-in` — The **CSR file** to be signed (`web-server.csr`)
- `-CA` — The **Intermediate CA’s certificate** (`inter.crt`)
- `-CAkey` — The **Intermediate CA’s private key** (`inter.key`)
- `-CAcreateserial` — Auto-generates `inter.srl` to track **serial numbers**
- `-outform DER` — Output format: **binary DER**, not PEM
- `-out` — Output file for the **signed certificate** (`web-server-der.crt`)
- `-days 825` — Certificate validity: **825 days**
- `-sha256` — Use **SHA-256** for secure, modern signing
- `-extfile` — Inject **X.509 extensions** from inline text

#### PEM vs DER

OpenSSL defaults to **PEM** format — it's **Base64-encoded** and human-readable in any text editor.  

**DER** is the **binary equivalent**, typically used in web servers and systems that require compact, machine-friendly formats. Both encode the same certificate data — only the outer wrapping differs.

#### Extensions Added During Signing

These extensions are injected using `-extfile` and are **critical** for browser recognition and trust:

- `keyUsage = digitalSignature, keyEncipherment`  
  Declares that the key is suitable for digital signing and key exchange.

- `extendedKeyUsage = serverAuth`  
  Specifies that this certificate is valid for **server authentication** (HTTPS).

- `subjectAltName = DNS:www.example.com, DNS:example.com`  
  Adds **Subject Alternative Names (SAN)** — **required by all modern browsers**.

## Inspecting the Signed Certificate

Once the certificate is signed, you can inspect its contents using:

```bash
openssl x509 -in web-server-der.crt -inform DER -noout -text
```

This command decodes the **DER-formatted certificate** and displays its structure in a human-readable form:

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            46:0d:9c:b0:fd:27:0f:6e:34:f2:b5:c3:60:c8:b0:6f:d4:77:23:b9
        # output truncated for brevity
```

Now you can verify everything — version, serial, issuer, subject, validity, extensions — all clearly laid out.

## Wrapping Up

You’ve generated a private key, created a CSR, had it signed by a CA, and inspected the final certificate — all the core steps of issuing a web server certificate.

_PKI doesn’t have to be a headache — and now you’ve seen why._
