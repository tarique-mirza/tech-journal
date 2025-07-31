# PKI With No Headache (Part 1): A Real World Example

Every time you visit a secure website — like [https://google.com](https://google.com) — your browser quietly checks whether the site’s certificate was issued by a trusted Certificate Authority (CA). One of the most widely used CA systems in the world is **Let’s Encrypt**, which has issued over 3 billion certificates and powers a massive portion of the internet.

Let’s Encrypt and other industry leaders like DigiCert and GlobalSign use a **two-tier certificate hierarchy** — a **Root CA** and one or more **Intermediate CAs**.  
In this post, we’re going to build that exact system — a minimal, fully working CA setup with the same two-tier hierarchy:

- A Root CA (**self-signed**)  
- An Intermediate CA (**signed by the root**)

> It’s a real, functional CA you can use to issue certificates — just like the big players do.  
We’ll use **OpenSSL** and basic **Linux commands** throughout this series.

---

## Step 1: Generate CA Private Key

Let’s begin by creating the private key for our Root CA:

```bash
openssl genrsa -out ca.key 2048
```

This generates `ca.key` — a PEM-formatted private key using the **PKCS#1** standard. It’s only about 1 KB in size, but it holds absolute power.

> If the private key of a real Root CA like Let’s Encrypt were ever leaked, over 3 billion certificates would become instantly untrusted — triggering a digital earthquake across the internet.

To prevent that kind of disaster, the Root CA is kept **offline** and used **only to sign Intermediate CA certificates**.  These intermediates handle day-to-day operations and remain online — keeping the root key locked away and safe.

---

## Step 2: Create the Root Certificate (X.509v3)

In the world of PKI, different entities use different types of certificates:

- **Root CA** — self-signed certificate that anchors trust
- **Intermediate CA** — signed by Root CA, used to issue certificates
- **Leaf Node (e.g., web server)** — signed by Intermediate, used by clients like browsers

### What’s in an X.509v3 certificate?

- Version (v3)
- Serial number
- Signature algorithm
- Issuer (who signed it)
- Validity (start and end dates)
- Subject (identity being certified)
- Public Key Info
- Extensions (KeyUsage, BasicConstraints, etc.)
- Signature (created using issuer’s private key)

Since this is a Root CA certificate, it will be **self-signed** — the **issuer** and **subject** are the same.

### Generate the Root Certificate

```bash
openssl req -x509 -new -key ca.key -out ca.crt -days 3650 -subj "/CN=PKIWNH Root CA"
```

**Breakdown:**

- `-x509` — generate a certificate instead of a CSR
- `-new` — create a new cert
- `-key ca.key` — use the root private key
- `-out ca.crt` — write to this file
- `-days 3650` — valid for 10 years
- `-subj` — subject name (CN only here)

---

## Step 3: Create the Intermediate CA

Now that we have our **Root CA** (`ca.key` + `ca.crt`), it’s time to create the **Intermediate CA** — the one that actually issues certificates.

This is how it works in real-world CA systems like Let’s Encrypt and DigiCert.  
The **Root stays offline**, and the **Intermediate** does all the signing. We’ll follow these steps:

1. Generate a private key  
2. Create a Certificate Signing Request (CSR)  
3. Sign the CSR with the Root CA

---

### Step 3.1: Generate Intermediate Private Key

```bash
openssl genrsa -out inter.key 2048
```

> This key will be used to sign **leaf certificates** (like web servers).

---

### Step 3.2: Create CSR (Certificate Signing Request)

```bash
openssl req -new -key inter.key -out inter.csr -subj "/CN=PKIWNH Intermediate CA"
```

> The CSR includes the **subject** and **public key**.  It will be **signed by the Root CA** to become a valid certificate.

---

### Step 3.3: Sign the CSR with Root CA

```bash
openssl x509 -req -in inter.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out inter.crt -days 1825
```

> This command creates `inter.crt` — a real **Intermediate CA certificate** valid for 5 years, signed by your **Root CA**.

**Explanation:**

- `-CAcreateserial` — creates a `ca.srl` file for serial tracking
- `-CA` and `-CAkey` — specify the Root CA used to sign
- `-in` and `-out` — input CSR and output certificate

---

At this point, you have a **working two-tier CA system**:

- `ca.crt` (Root CA) — used **only to sign Intermediate certificates**
- `ca.key` — must be kept **offline and secure**
- `inter.crt` + `inter.key` — used to sign **leaf certificates**

Just like Let’s Encrypt or DigiCert, your Root CA stays offline while your Intermediate CA goes live.

## Wrap Up

The only difference between your CA and a commercial one?

> Their Root CAs are **pre-installed** in billions of devices (browsers, OS, phones).  
> Yours is **not trusted yet**.

To use your CA in the real world, you’ll need to **manually install your Root CA certificate** on every system that needs to trust it — Windows, Android, Linux, or even just your browser.

---

**Next Up (Part 2):** We’ll dive into **X.509** — its format (PEM/DER), verification process, and key extensions like **Key Usage** and **SAN**.
