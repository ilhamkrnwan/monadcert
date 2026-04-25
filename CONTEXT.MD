# 📋 Task Context: Smart Contract Development

**Role:** KAIRAV (Backend / Smart Contract Engineer)
**Scope:** Fokus KETAT hanya pada pengembangan Smart Contract (Soulbound Token). Jangan mengambil alih atau mengubah bagian antarmuka/frontend yang menjadi tugas anggota tim lain (seperti Reyhan).

## 🎯 Target Objective
Fokus pengembangan dilakukan secara langsung melalui [Remix IDE](https://remix.ethereum.org/). Jangan buang waktu untuk konfigurasi atau *setup* lingkungan lokal (seperti Hardhat atau Foundry) agar dapat mempercepat proses *deployment*.

## 🛠️ Tugas Utama
1. **Implementasi Soulbound Token (SBT)**: 
   - Gunakan dan modifikasi standar **ERC-721**.
   - **WAJIB:** Matikan atau *override* fungsi transfer (misal: `transfer`, `transferFrom`, `safeTransferFrom`) sehingga setiap kali ada upaya untuk memindahkan token, transaksi akan selalu dibatalkan (*revert/error*). Token harus bersifat mutlak *non-transferable* yang terikat pada dompet penerima pertama.

## 🚀 Aksi (Deployment)
- Segera lakukan *deploy* kontrak yang telah diuji ke **Monad Testnet**.

## 📦 Output & Handoff
1. Ambil **Contract Address** dari kontrak yang berhasil di-*deploy*.
2. Ambil file **ABI (.json)** dari hasil kompilasi kontrak.
3. Kirimkan Contract Address dan ABI tersebut kepada **Reyhan**. 
4. Setelah penyerahan selesai, bantu tahap *testing* fungsionalitas.

---
**⚠️ ATURAN KETAT (STRICT RULES):**
- **DILARANG** mengubah, menyentuh, atau memberikan instruksi mengenai kode Frontend/UI.
- **FOKUS 100%** pada pembuatan, keamanan, dan fungsionalitas logika Smart Contract.
- Pastikan logika pemblokiran transfer token tidak memiliki celah (*loophole*).