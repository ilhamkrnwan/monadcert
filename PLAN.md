# 📐 MonadCert — MVP Scope, SRS & Implementation Plan

> **Role Scope (Kairav):** Smart Contract Engineer. Dokumen ini difokuskan pada logika on-chain dan interface kontrak yang dibutuhkan oleh tim frontend (Reyhan) dan backend/API (Albar/Heidar). Kairav TIDAK menyentuh UI/frontend.

---

## 1. 🎭 Definisi Aktor / Peran

| Aktor | Deskripsi |
|-------|-----------|
| **Provider Admin** | Super-admin protokol MonadCert. Bertugas memvalidasi dan mengelola daftar institusi yang diakui. |
| **Institution** | Universitas / komunitas / perusahaan yang sudah terverifikasi oleh Provider. Bisa menerbitkan sertifikat. |
| **Recipient (User)** | Individu yang menerima SBT ke wallet mereka. Mereka tidak bisa mentransfer token. |
| **Verifier (Third Party)** | HRD, pihak eksternal, atau siapapun yang ingin memverifikasi keaslian sertifikat (akses publik, tanpa login). |

---

## 2. 🔄 Alur Sistem (User Stories per Aktor)

### 2.1 Alur Pendaftaran Institusi (Community/Universitas)
```
Institusi mengisi form pendaftaran (nama, tipe: Univ/Corp/Community, alamat, wallet address)
  → Data disimpan di Supabase dengan status: "PENDING"
  → Notifikasi dikirim ke Provider Admin
  → Provider Admin mereview dan melakukan aksi approve/reject
    ↳ [APPROVE] → Provider Admin memanggil fungsi: grantInstitution(walletAddress) di Smart Contract
                → Status di Supabase diupdate: "APPROVED"
                → Institusi kini bisa login dan menerbitkan sertifikat
    ↳ [REJECT]  → Status di Supabase: "REJECTED" + alasan
```

> ⚠️ **Keputusan desain penting:** Apakah daftar institusi dikelola FULL on-chain (mapping di kontrak), atau hybrid (approval on-chain, data profil di Supabase)?
> **Rekomendasi:** Hybrid — hanya `mapping(address => bool) public isApprovedInstitution` yang on-chain. Profil nama/deskripsi di Supabase.

---

### 2.2 Alur Login
```
Institution Login:
  → Connect Wallet (MetaMask/RainbowKit)
  → Frontend memanggil isApprovedInstitution(walletAddress) di kontrak
    ↳ [true]  → Akses ke Institutional Dashboard diizinkan
    ↳ [false] → Ditolak / diarahkan ke halaman pendaftaran

User/Recipient Login:
  → Connect Wallet
  → Frontend memanggil balanceOf(walletAddress) / tokenOfOwnerByIndex
    → Menampilkan semua SBT yang dimiliki user di profil mereka
    
Provider Admin Login:
  → Connect Wallet
  → Frontend memverifikasi bahwa walletAddress == owner() di kontrak
    ↳ [true]  → Akses ke Admin Panel
```

> **Catatan:** Tidak ada sistem username/password tradisional. Auth murni berbasis wallet signature. Supabase digunakan hanya untuk storage data profil.

---

### 2.3 Alur Penerbitan Sertifikat (Minting)
```
[Institusi sudah login & verified]

1. Institusi membuka "Institutional Dashboard"
2. Input data penerima:
   - Nama lengkap penerima
   - Wallet address penerima
   - Judul sertifikat / program
   - Tanggal terbit
   - Metadata tambahan (opsional: IPFSHash dokumen PDF)
3. Metadata di-upload ke Supabase/IPFS → dapat tokenURI (URL/hash)
4. Institusi memanggil fungsi: issueCertificate(recipientAddress, tokenURI)
   → Kontrak memeriksa: require(isApprovedInstitution[msg.sender], "Not approved")
   → SBT di-mint ke wallet recipientAddress
   → Event Issued(tokenId, recipient, institution, timestamp) diemit
5. Recipient akan melihat SBT di wallet / profil mereka
```

---

### 2.4 Alur Verifikasi (oleh Pihak Ketiga)
```
[Verifier membuka halaman publik: /verify]

Opsi A — Via QR Code:
  → Scan QR → mendapat tokenId atau URL verifikasi
  → Frontend memanggil: getCredential(tokenId)
  → Kontrak mengembalikan: {issuedBy, recipient, tokenURI, isValid: true}
  → Ditampilkan: status VALID ✅ + detail sertifikat

Opsi B — Via Wallet Address:
  → Input wallet address penerima
  → Frontend memanggil: getCredentialsByOwner(address) [off-chain indexing]
    atau balanceOf() + tokenOfOwnerByIndex() untuk iterasi
  → Menampilkan semua sertifikat yang dimiliki address tersebut

Opsi C — Via Contract Address Langsung:
  → Input contract address + tokenId di block explorer
  → ownerOf(tokenId) mengembalikan pemilik
  → tokenURI(tokenId) mengembalikan metadata
```

---

### 2.5 Alur Provider Memvalidasi Institusi
```
[Provider Admin login ke Admin Panel]

1. Melihat daftar pendaftaran institusi dengan status PENDING
2. Mereview detail institusi (nama, bukti legalitas, dll) dari Supabase
3. Mengambil keputusan:
   ↳ APPROVE → Klik "Approve" → Frontend memanggil: grantInstitution(walletAddress)
              → Kontrak: isApprovedInstitution[walletAddress] = true
   ↳ REVOKE  → Klik "Revoke" → Frontend memanggil: revokeInstitution(walletAddress)  
              → Kontrak: isApprovedInstitution[walletAddress] = false
              → Semua SBT yang sudah diterbitkan institusi ini TETAP VALID
              (penerbitan baru tidak bisa dilakukan)
```

---

## 3. 🏗️ Arsitektur On-Chain vs Off-Chain

| Data | Lokasi | Alasan |
|------|--------|--------|
| Status approved institusi | **On-chain** | Trustless, cek langsung dari kontrak |
| SBT ownership | **On-chain** | Permanen, tidak bisa dipalsukan |
| tokenId → metadata URI | **On-chain** | Standar ERC-721 tokenURI |
| Profil detail institusi (nama, deskripsi, logo) | **Supabase** | Data yang bisa diupdate, tidak perlu gas |
| Metadata sertifikat lengkap (PDF, detail) | **IPFS + Supabase** | File besar, hanya hash-nya on-chain |
| Riwayat minting per institusi | **Event logs + Supabase** | Indexing off-chain untuk query cepat |

---

## 4. 📋 SRS — Smart Contract Functions (Scope Kairav)

### Contract: `EduTrust.sol`
Inherit dari: `ERC721`, `Ownable` (OpenZeppelin)

```solidity
// === STATE VARIABLES ===
mapping(address => bool) public isApprovedInstitution;
mapping(uint256 => address) public issuedBy;      // tokenId → institution wallet
uint256 private _tokenIdCounter;

// === EVENTS ===
event InstitutionGranted(address indexed institution);
event InstitutionRevoked(address indexed institution);
event CertificateIssued(uint256 indexed tokenId, address indexed recipient, address indexed institution);

// === ADMIN FUNCTIONS (onlyOwner) ===
function grantInstitution(address institution) external onlyOwner;
function revokeInstitution(address institution) external onlyOwner;

// === INSTITUTION FUNCTIONS ===
function issueCertificate(address recipient, string calldata tokenURI) external returns (uint256 tokenId);
  // require: isApprovedInstitution[msg.sender]

// === SOULBOUND — OVERRIDE TRANSFER ===
function transferFrom(...) → revert("SBT: Non-transferable")
function safeTransferFrom(...) → revert("SBT: Non-transferable")
function safeTransferFrom(..., bytes) → revert("SBT: Non-transferable")
function approve(...) → revert("SBT: Non-transferable")
function setApprovalForAll(...) → revert("SBT: Non-transferable")

// === READ FUNCTIONS (PUBLIC) ===
function getCredential(uint256 tokenId) external view returns (address recipient, address institution, string memory uri);
```

---

## 5. 🗓️ Implementation Plan (Bertahap)

### Phase 0 — Diskusi & Persetujuan (SEKARANG)
- [ ] Review dan setujui dokumen ini
- [ ] Klarifikasi: apakah perlu fungsi **revoke certificate** (membakar/invalidasi SBT)?
- [ ] Klarifikasi: apakah satu institusi bisa punya **multiple admin wallet**, atau cukup satu?
- [ ] Klarifikasi: metadata on-chain minimal apa yang dibutuhkan Reyhan (frontend)?

### Phase 1 — Smart Contract (Kairav)
- [ ] Buat `contracts/EduTrust.sol` menggunakan OpenZeppelin ERC-721
- [ ] Implementasi Soulbound logic (override transfer functions)
- [ ] Implementasi institution approval system
- [ ] Implementasi `issueCertificate()`
- [ ] Compile & test di Remix IDE
- [ ] Deploy ke Monad Testnet (Chain ID: 10143, RPC: `https://testnet-rpc.monad.xyz`)
- [ ] Verifikasi kontrak via API (MonadVision, Socialscan, Monadscan sekaligus)
- [ ] Kirim Contract Address + ABI ke Reyhan

### Phase 2 — Off-chain Backend (Albar/Heidar)
- [ ] Setup Supabase tables: `institutions`, `certificates`, `pending_approvals`
- [ ] API endpoint untuk form pendaftaran institusi
- [ ] Integrasi dengan event logs kontrak untuk indexing

### Phase 3 — Frontend Integration (Reyhan)
- [ ] Wallet connect (RainbowKit + Wagmi)
- [ ] Institutional Dashboard (minting form)
- [ ] Public verification page
- [ ] Admin panel (approve/revoke institutions)

---

## 6. ❓ Pertanyaan yang Perlu Dijawab Sebelum Eksekusi

1. **Revoke SBT?** — Apakah sertifikat yang sudah terbit bisa dibatalkan (burn) oleh institusi atau admin? Atau bersifat permanen selamanya?
2. **Multi-admin institusi?** — Satu institusi = satu wallet, atau satu institusi bisa punya tim dengan beberapa wallet yang bisa minting?
3. **Batch minting?** — Apakah perlu fungsi `batchIssueCertificate(address[] recipients, string[] uris)` untuk efisiensi saat wisuda?
4. **Metadata standard?** — Pakai JSON standard ERC-721 (`name`, `description`, `image`) atau custom schema?
5. **IPFS atau Supabase URL?** — tokenURI mengarah ke mana? IPFS (desentralisasi permanen) atau Supabase (bisa diupdate)?

---

> **Rekomendasi Saya:** Implementasikan **batch minting** dari awal karena itu adalah keunggulan utama MonadCert di Monad (parallel execution). Ini juga yang membedakan proyek ini dari solusi serupa.
