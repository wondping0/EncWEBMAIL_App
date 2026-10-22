<div align="center">

# EncWEBMAIL_App

**A web-mail prototype that encrypts attachments end-to-end with passphrase-derived RSA keys and AES.**

![Node.js](https://img.shields.io/badge/Node.js-Express-38E1FF?style=flat-square&labelColor=0A1428&logo=nodedotjs&logoColor=38E1FF)
![React](https://img.shields.io/badge/React-TypeScript-38E1FF?style=flat-square&labelColor=0A1428&logo=react&logoColor=38E1FF)
![MySQL](https://img.shields.io/badge/MySQL-Sequelize-38E1FF?style=flat-square&labelColor=0A1428&logo=mysql&logoColor=38E1FF)

</div>

---

## Overview

Users log in, send messages with one attachment to another user's e-mail address, and read the messages they received or sent.
Each message has a classification: **Biasa** (normal), **Rahasia** (secret) or **Sangat Rahasia** (top secret).
For anything other than *Biasa*, the attachment is encrypted in the browser before upload, and only the receiver's passphrase can decrypt it.

## Encryption scheme

All cryptography runs in the frontend (`CreateMessagePage.tsx`, `DetailMessagePage.tsx`). The backend only stores the results.

**Key pairs from passphrases.** Each user's RSA-2048 key pair is generated with `node-forge`, using a PRNG seeded with the user's passphrase.
The same passphrase always reproduces the same key pair, so the private key is never stored anywhere. The server keeps only:

- the public key PEM (`be/keystore/public/<userId>_public.key`);
- a bcrypt hash of the passphrase.

Changing the passphrase regenerates and overwrites the public key.

**Sending (Rahasia / Sangat Rahasia):**

1. The browser generates a random 16-character secret key.
2. The attachment is encrypted with CryptoJS AES, using `SHA-256(secret)` as the passphrase. The result is stored as `ciphertext_iv_salt` and uploaded as `enc_<filename>`.
3. The secret is encrypted with the receiver's RSA public key (fetched from `/api/message/getpublic`) and saved with the message as `encKey`.
4. The SHA-256 of the original file is saved as `fileHash`.

**Receiving:**

1. The receiver enters their passphrase, and the server checks it against the bcrypt hash.
2. The browser regenerates the RSA private key from the passphrase and decrypts `encKey`.
3. It downloads the attachment and AES-decrypts it locally.

*Biasa* attachments are uploaded and downloaded as-is. The message text itself is stored as entered; the encryption covers the attachment.

## Project layout

| Path | Description |
| --- | --- |
| `be/` | Backend: Express API, Sequelize models (MySQL), JWT auth |
| `be/server.js` | App entry: routes, static `/attachment/get` for uploaded files, optional `initial()` seeding of demo users |
| `be/routes/`, `be/controllers/` | `auth` (sign-in, change/check passphrase) and `message` (create, delete, upload attachment, list received/sent, get public key) |
| `be/models/` | `users`, `messages`, `attachments` and the `attachments_message` join table |
| `be/keystore/public/` | Users' public keys |
| `be/app.js` | Standalone script that writes a random RSA-2048 key pair to `keystore/private` and `keystore/public` |
| `fe_backup/` | Frontend: React + TypeScript (Create React App), MUI, Redux Toolkit, axios, crypto-js, node-forge |
| `fe` | Recorded as a submodule pointer with no `.gitmodules` entry, so its contents are not part of this repo. The frontend source is in `fe_backup/` |

### API

| Method | Route | Auth |
| --- | --- | --- |
| POST | `/api/auth/signin` | no |
| POST | `/api/auth/changepassphrase` | JWT |
| POST | `/api/auth/checkpassphrase` | JWT |
| POST | `/api/message/create` | JWT |
| POST | `/api/message/delete` | JWT |
| POST | `/api/message/attachment/new` (multipart `file`) | JWT |
| GET | `/api/message/draft/get` (messages received) | JWT |
| GET | `/api/message/sent/get` | JWT |
| POST | `/api/message/getpublic` | JWT |

## Requirements

- Node.js and npm
- MySQL

## Usage

### Backend

```bash
cd be
npm install
```

Set the variables read by `configs/` in `be/.env`:

- `SECRET_TOKEN` (JWT signing secret)
- `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`

Create the database. On the first run, uncomment `initial()` in `server.js` to create the demo users and their public keys. Then start the server:

```bash
npm start            # nodemon server.js
```

The server listens on `localhost:$HTTP_PORT`, defaulting to `8080`, which is the address the frontend expects (`fe_backup/src/configs/listOfUrl.tsx`).

### Frontend

```bash
cd fe_backup
npm install
npm install node-forge   # imported by the message pages but not listed in package.json
npm start
```

---

<div align="center">

Part of [wondping0](https://github.com/wondping0)'s cryptography notebooks · [wondping0.github.io](https://wondping0.github.io)

</div>
