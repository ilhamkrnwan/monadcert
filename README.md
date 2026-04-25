# 🎓 MonadCert: Decentralized Credential Protocol

MonadCert adalah protokol verifikasi ijazah dan sertifikat berbasis **Soulbound Tokens (SBT)** yang dibangun di atas jaringan [Monad](https://monad.xyz/). Proyek ini dirancang untuk mengatasi masalah pemalsuan dokumen akademik dan profesional dengan memanfaatkan kecepatan eksekusi paralel Monad untuk pencetakan (minting) massal yang instan dan hemat biaya.

## 🚀 Mengapa MonadCert?

Di ekosistem Web3 saat ini, biaya gas dan _latency_ menjadi penghambat utama bagi institusi besar (seperti universitas) untuk mengadopsi blockchain.

- **Parallel Execution**: MonadCert mampu menangani ribuan penerbitan sertifikat secara simultan saat musim kelulusan tanpa _network congestion_.
- **Immutable & Soulbound**: Sertifikat diterbitkan sebagai SBT yang tidak dapat dipindahtangankan, memastikan identitas digital tetap melekat pada pemilik aslinya.
- **Universal Protocol**: Bisa digunakan oleh Universitas (Ijazah), Perusahaan (Sertifikat Magang), maupun Komunitas (Sertifikat Workshop).

## ✨ Fitur Utama

- 🏢 **Institutional Dashboard**: Portal bagi institusi atau komunitas terverifikasi untuk mengunggah metadata dan melakukan _batch-minting_.
- 🔍 **Instant Verification**: HRD atau pihak ketiga dapat memverifikasi keaslian dokumen hanya dengan memindai kode QR atau memasukkan _Wallet Address_.
- 🗄️ **Hybrid Storage**: Metadata dengan kapasitas besar disimpan di Supabase & IPFS, sementara bukti kriptografis (Hash) diamankan di dalam Monad Smart Contract.
- 📱 **Mobile-Ready**: Terintegrasi dengan Capacitor untuk akses verifikasi cepat melalui _smartphone_.

## 🛠️ Tech Stack

- **Blockchain**: Monad (EVM-Compatible)
- **Smart Contracts**: Solidity (SBT Implementation)
- **Frontend**: Nuxt 4, TypeScript, Tailwind CSS
- **Backend & Database**: Supabase (Auth & Metadata Registry)
- **Mobile**: Capacitor
- **Dev Tools**: Foundry / Hardhat

## 🏗️ Alur Kerja (Workflow)

1. **Vetting**: Institusi mendaftarkan diri dan divalidasi oleh admin protokol.
2. **Issuance**: Institusi menginput data penerima (Nama, NIM, Jurusan/Topik).
3. **Minting**: Smart contract mencetak SBT ke _wallet_ penerima di jaringan Monad.
4. **Display**: Pengguna melihat sertifikat di profil "EduTrust" mereka.
5. **Validation**: Pihak ketiga melakukan pengecekan status `isVerified` langsung ke smart contract.

## 📦 Pengembangan (Local Setup)

```bash
# Clone repository
git clone https://github.com/username/monad-cert.git
cd monad-cert

# Install dependencies
npm install

# Setup Environment Variables (.env)
# Pastikan Anda menambahkan variabel berikut:
# MONAD_RPC_URL="..."
# SUPABASE_KEY="..."

# Jalankan development server
npm run dev
```

## 📝 Smart Contract Gist (SBT Logic)

Logika inti pada smart contract untuk menjadikan token tidak dapat dipindahtangankan (_Soulbound_):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Logika inti: Fungsi transfer dinonaktifkan untuk menjadi Soulbound
function _update(address to, uint256 tokenId, address auth) internal override returns (address) {
    address from = _ownerOf(tokenId);
    if (from != address(0) && to != address(0)) {
        revert("MonadCert: SBT is non-transferable");
    }
    return super._update(to, tokenId, auth);
}
```

## 👤 Author

- **Albar**
- **Heidar**
- **Ilham**
