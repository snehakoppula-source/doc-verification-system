# Doc Verification System

A secure document verification system built with the MERN stack (MongoDB, Express.js, React.js, Node.js) and cryptographic techniques to ensure authenticity of uploaded documents. This project allows institutes to upload certificates and companies to verify their authenticity in a trustless manner.

## Table of Contents

- [Features](#features)

- [Tech Stack](#tech-stack)

- [Frontend](#frontend)

- [Installation](#installation)

- [Usage](#usage)

- [API Endpoints](#api-endpoints)

- [Project Structure](#project-structure)

- [License](#license)

### Features

- Institute Uploads: Institutes can securely upload certificates in PDF format.

- Company Verification: Companies can verify if a certificate is authentic.

- Cryptographic Verification: PDF integrity is verified using hashes, digital signatures.

- Role-based Access: User roles (Institute / Company / Admin) control access.

- MERN Stack: Modern, scalable architecture.

### Tech Stack

- Frontend: React 19, Vite, React Router, Axios, and a hand-written CSS design system (no UI framework)

- Backend: Node.js, Express.js, MongoDB, Mongoose

- Security: Crypto (RSA, SHA-256) for document verification

- Deployment: Docker (optional), Node.js scripts

### Frontend

The interface is organised around what each role needs to do:

| Route | Who | What it does |
| ----- | --- | ------------ |
| `/` | Everyone | Landing page: how verification works, possible results, security specifications, FAQ |
| `/register`, `/login` | Everyone | Role picker, inline validation, show/hide password. `/register?role=Company` pre-selects a role |
| `/company/verify` | Company | Enter institution code and roll number, drop in the PDF, watch the check run, read a result report (printable) |
| `/institute/upload` | Institution | Register the public key, then upload signed PDFs with the signatures JSON in batches, with per-file status |
| `/dashboard` | Individual | Account summary and pointers to the company and institution roles |

Notes for development and your viva:

- No API contract was changed. Every request uses the same endpoints and payloads as before (`/login`, `/register`, `/company/verify`, `/institute/upload`, `/institute/certificate`).
- The product name is set in one place: `frontend/src/config.js`. Change `APP_NAME` to rename it everywhere.
- The guilloche rosettes (logo, hero, result seals) are generated from hypotrochoid curves in `src/components/Guilloche.jsx`, so they are code, not image files.
- The verification page shows the staged checks (record found, key on file, signature valid) by interpreting the API's status codes and messages. If you change those backend messages, update `interpret()` in `src/pages/CompanyVerify.jsx`.
- The SHA-256 shown on a result is computed in the browser from the file you selected. The backend verifies the RSA signature itself and does not return a hash.
- Fonts (Newsreader, Public Sans) load from Google Fonts in `index.html`. Offline, the browser falls back to Georgia and the system font.
- Additions: the institution page refuses private keys and non-PEM files before upload, and the `/dashboard` route that `User` accounts were already redirected to now exists.

### Installation

```bash

# Clone the repository
   git clone https://github.com/RSaiKartik2202/doc-verification-system.git
   cd doc-verification-system
# Backend setup
   cd backend
   npm install
# Frontend setup
   cd ../frontend
   npm install
# Environment variables
   MONGO_URI=<your_mongodb_uri>
   JWT_SECRET=<your_jwt_secret>`
   PORT=5000
# Run the app
   cd backend
   npm run dev
   cd ../frontend
   npm start/npm run dev
```

### Usage

1. Institute Flow:

   - Sign up as an institute.

   - Upload Public Key Certificate (PKC).

   - Upload digitally signed student certificates.

   - Upload digital signatures of certificates.

   - Certificates are hashed and stored in the database along with the digital signature.

2. Company Flow:

   - Sign up as a company.

   - Submit a certificate PDF for verification.

   - System checks the hash against stored certificates digital signature and returns verification status.

3. Digital Signature Workflow:

   1. Key Generation

   - Run scripts/generateKeys.js to generate a public/private key pair.

   - The private key is used by the institute to sign certificates.

   - The public key will be used later for verification by companies.

   2. Signing Certificates

   - Place all certificate PDFs in data/Certificates.

   - Run scripts/signFiles.js to generate a digital signature for each certificate using the private key.

   - The script saves the signatures in data/signatures/ and creates a JSON file containing:

   - Certificate file name

   - Digital signature

   3. Submission by Institutes

   - Institutes must submit this JSON file along with the uploaded certificates.

   - The JSON is used by the backend to verify certificate authenticity when a company requests verification.

### API Endpoints

| Method | Endpoint                   | Description                                   |
| ------ | -------------------------- | --------------------------------------------- |
| POST   | /api/auth/register         | Register user                                 |
| POST   | /api/auth/login            | Login user                                    |
| POST   | /api/institute/upload      | Upload a certificate(PDF)                     |
| POST   | /api/company/verify        | Verify a certificate(PDF)                     |
| POST   | /api/institute/certificate | Submit a PKC(Public key certificate)          |
| GET    | /api/institute/certificate | Get the submitted PKC(Public key certificate) |

### Project Structure

``` bash
doc-verification-system/
├─ backend/
│ ├─ controllers/
│ ├─ models/
│ ├─ routes/
│ ├─ utils/
│ ├─ server.js
├─ frontend/
│ ├─ src/
│ │ ├─ components/   # Navbar, Footer, Dropzone, Guilloche (logo + seals), Icon
│ │ ├─ pages/        # Home, Login, Register, CompanyVerify, InstituteUpload, Dashboard
│ │ ├─ context/      # AuthContext
│ │ ├─ styles/       # design tokens + page styles
│ │ ├─ utils/        # axios instance, formatting helpers, role routing
│ │ ├─ config.js     # product name
│ │ ├─ App.jsx
│ │ └─ main.jsx
├─ data/
│ ├─ certificates/ # Folder to store uploaded certificate PDFs
│ ├─ keys/ # Folder containing public/private key pairs
│ └─ signatures/ # Folder to store digital signatures for certificates
├─ scripts/
│ ├─ generateKeys.js # Script to generate public/private key pairs
│ └─ signFiles.js # Script to digitally sign certificate files and save into JSON
├─ README.md
└─ package.json
```
### License

This project is licensed under the MIT License.
```
