![CampusCred Banner](frontend/static/images/banner.png)

A blockchain-based credential verification system for educational institutions, built with Flask (backend), Hardhat (smart contracts), and a vanilla JS frontend.

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Running the Application](#running-the-application)
- [End-to-End Demo Flow](#end-to-end-demo-flow)
- [Testing](#testing)
- [Smart Contract and Blockchain](#smart-contract-and-blockchain)
- [Selective Disclosure and Evidence Signing](#selective-disclosure-and-evidence-signing)
- [Troubleshooting](#troubleshooting)
- [Team](#team)
- [License](#license)

## Overview

CampusCred enables educational institutions to issue verifiable digital credentials as non-transferable NFTs on the Ethereum Sepolia testnet.

**Student Portal** -- Submit credential claims (micro-credential, course completion, diploma) with supporting evidence (PDF, images, docs). Evidence is stored privately (local or S3) with a SHA-256 hash for integrity.

**Instructor Dashboard** -- Review and approve or reject pending claims. On approval, credential metadata is uploaded to IPFS (via Pinata, or a deterministic mock hash if not configured) and a non-transferable CampusCred NFT is minted to the student's wallet. The dashboard tracks statistics: total claims, pending, approved, and minted.

**Verification System** -- A public page to verify credentials by Token ID, requiring no wallet. Shows issuer, course, credential type, issuance date, on-chain transaction, and metadata. Uses on-chain data (`ownerOf`, `tokenURI`, `isRevoked`) alongside the local DB.

**Selective PII Disclosure** -- Credential owners can generate a time-limited verifier link (15 minutes). A recruiter following the link can see the legal name, email, and evidence file name, and download a digitally signed PDF of the evidence.

**Blockchain Integration** -- A Solidity smart contract (`CampusCredNFT`, non-transferable ERC-721) deployed to Sepolia. Minting is performed from a backend-controlled deployer wallet. See `DEPLOYMENT.txt` for deployment details.

### Current Status (Final Prototype)

The end-to-end prototype is fully implemented:

- Student submission with file upload and private storage
- Instructor approval and NFT minting on Sepolia
- Public verification by token ID (no wallet required)
- Time-limited verifier links for PII and signed PDF download
- Hardhat-based contract deployment and tests
- Full E2E browser testing with mocked MetaMask injection

## Project Structure

```
campuscred-nft-system/
├── DEPLOYMENT.txt              # Sepolia deployment info (address, date)
├── README.md
├── backend/
│   ├── app/
│   │   ├── __init__.py         # Flask app factory
│   │   ├── config.py           # Configuration loader (env-based)
│   │   ├── models.py           # SQLAlchemy models (Claim)
│   │   ├── routes/             # Blueprints (auth, claims, instructor, verify)
│   │   └── services/           # Logic (blockchain, storage, ipfs, signer)
│   ├── e2e/                    # End-to-end Playwright tests
│   │   ├── __init__.py
│   │   ├── conftest.py         # Live server fixture and blockchain mocks
│   │   ├── mocks.py            # window.ethereum injection
│   │   └── test_full_flow.py   # Full critical path tests
│   ├── tests/                  # Backend unit tests (pytest)
│   ├── .env.example            # Backend environment template
│   ├── requirements.txt        # Python dependencies
│   ├── run.py                  # Flask entry point
│   └── setup_database.py       # DB initialisation helper
├── frontend/
│   └── static/                 # CSS, JS, and images
├── contracts/
│   ├── CampusCredNFT.sol       # Solidity credential NFT contract
│   └── CampusCredNFT_ABI.json  # ABI exported for backend Web3
├── test/
│   └── CampusCredNFT.test.js   # Hardhat/Mocha tests for the contract
├── ignition/                   # Hardhat Ignition deployment modules
├── scripts/                    # Hardhat helper scripts (deploy, export ABI)
├── hardhat.config.ts           # Hardhat 3 configuration (TypeScript)
└── package.json                # Node/Hardhat dev dependencies
```

## Features

### Student

- Submit claims with name, email, course code, and credential type.
- Upload evidence files securely (stored off-chain).
- Connect a wallet via MetaMask to associate credentials with an Ethereum address.
- View status of submitted claims (Pending, Approved, Minted).

### Instructor

- Authenticated via a specific instructor wallet address.
- Dashboard view of all pending claims.
- Approve claims to trigger metadata generation, IPFS pinning, and blockchain minting.
- Reject claims with a recorded reason.

### Verifier / Recruiter

- **Public verification:** Enter a Token ID to view the immutable blockchain record (issuer, date, course).
- **Private link:** The credential owner generates a 15-minute link. The verifier can download digitally signed evidence and view PII (name and email).

## Prerequisites

- Python 3.12+
- Node.js 18+
- Git
- MetaMask (for manual testing)
- Playwright browsers (for E2E testing)

## Installation

### 1. Clone the Repository

```bash
git clone <repository-url>
cd campuscred-nft-system
```

### 2. Backend Setup

```bash
cd backend
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# Install dependencies (Flask, Web3, Playwright)
pip install -r requirements.txt

# Install Playwright browsers for E2E testing
python -m playwright install chromium

# Create .env and initialize DB
cp .env.example .env
python setup_database.py
```

### 3. Smart Contract Setup

```bash
npm install
```

## Running the Application

### Backend (Flask API + HTML UI)

```bash
cd backend
source venv/bin/activate
python run.py
```

Access the app at http://localhost:5000.

### Smart Contracts (Hardhat)

```bash
# Start local node (optional)
npx hardhat node
```

## End-to-End Demo Flow

1. **Student:** Go to `/student/portal`. Fill out the form, upload a PDF, and submit.
2. **Instructor:** Connect wallet (use the address in `auth.py`). Go to `/instructor/dashboard`. Click Approve.
3. **System:** Uploads metadata to IPFS, mints NFT on Sepolia, updates DB.
4. **Verifier:** Go to `/verify/`, enter Token ID, see valid credential.
5. **Private Share:** Student generates link, verifier downloads signed PDF.

## Testing

This project uses a testing pyramid strategy with unit, integration, and end-to-end browser tests.

### Backend Unit Tests

Cover models, routes, storage, and services logic using pytest.

```bash
cd backend
python -m pytest tests/
```

### Smart Contract Tests

Cover minting, role-based access control, and revocation using Hardhat and Chai.

```bash
npx hardhat test
```

### End-to-End Tests

Cover the full lifecycle (student, instructor, verify) using Playwright.

The test setup runs the Flask server in a background thread using an in-memory DB, launches a headless Chromium browser, injects a fake `window.ethereum` provider to mock MetaMask, and patches `BlockchainService` to simulate minting without waiting for Sepolia confirmation.

```bash
cd backend

# Run in headed mode to see the browser actions
python -m pytest e2e/test_full_flow.py --headed
```

## Smart Contract and Blockchain

The contract `CampusCredNFT.sol` is an ERC-721 deployed to the Sepolia testnet.

Key properties:

- `MINTER_ROLE`: Only the backend deployer wallet can mint.
- `revoke()`: Allows the university to invalidate credentials.
- **Non-transferable:** Overrides `_update` to prevent transfer or sale of credentials (soulbound).

### Deployment (Sepolia)

To (re-)deploy with Hardhat:

```bash
npx hardhat run scripts/deploy.js --network sepolia
```

## Selective Disclosure and Evidence Signing

### Verifier Links

Time-limited tokens (15 minutes) are stored in backend memory and can only be generated by the wallet owner of the credential.

### PDF Signing

`PDFSignerService` generates a self-signed PKI certificate on the fly (stored in `private_storage`). When a recruiter downloads evidence via a private link, the PDF is digitally signed to prove it came from the CampusCred system.

## Troubleshooting

### "Playwright: Executable doesn't exist"

If E2E tests fail with browser errors, the binary installation might be corrupted. Force a clean reinstall:

```bash
rm -rf ~/Library/Caches/ms-playwright
python -m playwright install chromium
```

### "ImportError: cannot import name 'ContractName' from 'eth_typing'"

This is a dependency conflict between web3.py and newer versions of eth-typing. Fix by pinning the version:

```bash
pip install "eth-typing<5.0.0"
```

### "Database is locked"

Make sure you don't have the SQLite file open in another viewer or IDE while running tests or the server.

### "ModuleNotFoundError: No module named 'flask'"

Your virtualenv is not active. Run `source venv/bin/activate`.

## Team

**Group 10** -- Fall 2025, Technical University of Denmark (DTU)

**Course:** Software Processes and Patterns (02369)

This system is a prototype for educational purposes and does not represent an official DTU credentialing system.

## License

This project is part of academic coursework at DTU. Usage is limited to educational and demonstration purposes.