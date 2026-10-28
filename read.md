# NONCE Genesis: langkah sampai challenge ACTIVE

Catatan dari run nyata (BSC mainnet, chain 56, Binance Agentic Wallet dengan Agent ID `.ai`).
Challenge berhasil sampai **ACTIVE** dua kali. Dua-duanya gagal di tahap **ambil fragmen**, karena server MCP overload (429/502).
Solver bukan masalahnya: BP q-hidden (lihat file `3SAT`) menyelesaikan instance uji 1536 var × 33.792 klausa dalam < 1 detik.

Jangan pernah taruh token MCP, signature, atau capability di log/chat/repo.

## Alamat resmi (cek ulang ke https://nonceprotocol.xyz/api/mint-prompt)
- MCP: `https://mcp.nonceprotocol.xyz/mcp` (Streamable HTTP, JSON-RPC)
- Auth: `POST https://mcp.nonceprotocol.xyz/auth/challenge` lalu `POST .../auth/verify`
- FreeMint `0x7538ba8C1A82E9706dB6d2463C15f69ADc01a17e`
- NFT `0x053259A3D30E63d9447E3d90813C6a494bcB398a`
- AgentGate `0xc8D8EebAd7Fdd590edA6d23164B1eC014913Ce8f`
- Registry `0x60B9262456dE011F7d15a54CdD8c48bAaa8B022F`

## 0. Setup wallet (sekali)
```bash
npx skills add binance/binance-skills-hub --skill binance-agentic-wallet
npm i -g @binance/agentic-wallet        # CLI `baw`
baw wallet status --json                 # harus CONNECTED
baw auth signin && baw auth verify       # kalau belum: kode pairing dikonfirmasi di Binance App
baw wallet address --json                # alamat chain 56
baw wallet settings --json               # devMode.enabled harus true
```
Agent ID = `handleOf(wallet)`, nilainya dikirim sebagai **string desimal**. Pastikan `AgentGate.isAuthorized(agentId, wallet)` bernilai true.

## 1. Client MCP
Setiap request wajib punya header:
```
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2025-06-18
User-Agent: <isi apa saja>          # tanpa UA sempat kena 403
Authorization: Bearer <token>       # khusus tool yang protected
Mcp-Session-Id: <dari response initialize>
```
Urutannya `initialize`, lalu notifikasi `notifications/initialized`, lalu `tools/call`.
Response bisa berupa SSE (`data: {...}`); ambil baris `data:` yang terakhir.
Hasil tool ada di `result.content[].text` (berisi JSON string). Selalu cek `result.isError`, karena HTTP 200 belum tentu sukses.
Kalau dapat HTTP 404 padahal sudah punya session id, berarti session basi: initialize ulang.

**Penting:** pakai **satu proses dan satu session MCP** dari prepare sampai submit. Handshake ulang di tengah jalan memakan menit saat server overload.

## 2. Cek admission (tanpa token)
`protocol_info({})`: lanjut hanya kalau `admission="open"`, mintPrice "0", failureFee "0", maxMintsPerWallet 2, dan `signer` sama dengan `FreeMint.signer()` on-chain.

## 3. Login MCP (token berlaku 1 jam)
1. `POST /auth/challenge` dengan body `{"wallet":"0x..."}`
2. Cek `typedData`: domain `NONCE MCP Sign In`, version `1`, chainId 56, primaryType `SignIn`, wallet = wallet kita, audience `mcp.nonceprotocol.xyz`, statement "Sign in only...". Challenge-nya kedaluwarsa dalam ±5 menit.
3. Preview, lalu execute:
```bash
baw sign-message preview --binanceChainId 56 --signType EIP712 --message "$(cat walletRequest.json)" --json
baw sign-message execute --requestId <requestId dari preview> --json
```
   Yang dikirim adalah **seluruh `walletRequest`**, bukan cuma typedData.
4. `POST /auth/verify` dengan body `{wallet, challenge, signature, signatureRecovery?}`, lalu simpan `token` secara privat (chmod 600).

**Timing:** sebelum `start_attempt`, sisa umur token harus ≥ 900 dtk + 5 menit. Jadi login MCP **tepat sebelum** prepare, jangan dari awal.

## 4. prepare_attempt
```json
{"wallet":W,"beneficiary":W,"agentId":"<desimal>","idempotencyKey":"<uuid baru>"}
```
- Kalau dapat `SERVICE_UNAVAILABLE` atau 429, retry dengan **idempotencyKey yang sama**. Jangan bikin key baru, nanti attempt-nya dobel.
- Response berisi `attemptId`, `expiresAt` (≈10–15 menit), `benchmark`, dan `startWalletRequest`.
- Contoh benchmark nyata: `nonce-logic-balanced-1536`, 1536 variabel, 33.792 klausa, 5 literal per klausa, 32 fragmen, toolBudget 96, deadline 900 dtk sejak start sukses.
- **Attempt yang sudah di-prepare tapi tidak di-start sebelum `expiresAt` hangus** (`EXPIRED`, tanpa biaya). Ini terjadi 2× karena `start_attempt` tertahan 429.

## 5. Tanda tangan StartAuthorization lalu start_attempt
- Cek domain `NONCE Free Mint`, version 1, chainId 56, verifyingContract = FreeMint, primaryType `StartAuthorization`, `message.wallet` = wallet kita, `message.attemptId == keccak256(utf8(UUID))`.
- Preview dan execute `startWalletRequest` lewat `baw sign-message`, sama seperti langkah 3.
- `start_attempt({attemptId, signature, signatureRecovery?})` mengembalikan `state:"ACTIVE"`, `deadline`, dan `capability` (rahasia).
- Kalau gagal transient, baca `attempt_status({attemptId})`, lalu retry start dengan signature yang sama. Jangan prepare ulang.

Tips: siapkan script yang terus menunggu file signature. Begitu signature ada, `start_attempt` langsung terkirim lewat session yang masih terbuka. Dengan cara ini start tembus dalam ±3 menit walau server 429.

## 6. Fragmen (di sini macetnya)
`inspect_fragment({attemptId, capability, fragmentId})` untuk 0..31. Setiap fragmen berisi 1056 klausa (`firstClause`, `totalClauses`, `variables`, `artifactFormat`, `remainingReads`).
- `fragment.clauses` kadang datang sebagai **JSON string**, kadang sebagai list. Parse dua-duanya.
- Hasil run: attempt #3 dapat 8/32 fragmen dalam 15 menit, attempt #4 cuma 1/32 (lebih dari 120 balasan 429/502). Kalau server overload, satu fragmen bisa butuh 1–8 menit.
- Ide yang **belum dicoba** (cek dulu aturan/ToS-nya):
  - Mulai attempt hanya saat server sepi. Probe `initialize` tanpa token tiap 30 dtk; kalau mayoritas OK, baru login dan prepare.
  - Cek apakah server menerima beberapa `tools/call` sekaligus dalam satu session (JSON-RPC batch), atau request paralel, karena masih ada toolBudget 96 > 32. Kami sengaja tidak mencoba karena policy prompt meminta satu request per waktu saat recovery.
  - Hormati header `Retry-After` kalau ada.

## 7. Solve dan submit (sekali)
- Validasi: 32 fragmen, total klausa = benchmark, semua klausa berisi 5 literal, |literal| antara 1 dan 1536.
- Solver: BP q-hidden (q=0.618, bobot `q^(jumlah literal benar)`, damping 0.5, sampai 400 iterasi, round tanda medan). Kalau masih ada sisa klausa gagal, lanjutkan WalkSAT dari hasil BP. Lihat `3SAT` dan `solver2`.
- Cek ulang **semua** klausa sendiri, lalu panggil `submit_artifact({attemptId, capability, artifact:{assignment:"<1536 char 0/1>"}, nonce:"0"})`. Karakter ke-k (0-based) adalah variabel k+1. Format `artifact` disesuaikan dengan `artifactFormat` dari fragmen.
- Kalau hasilnya `VERIFIED_WIN`: panggil `mint_transaction({attemptId})`, verifikasi authorization, lalu `baw contract-call preview --binanceChainId 56 --from W --to FreeMint --value 0 --inputData <calldata> --json`, lalu execute. Terakhir cek receipt (event `Minted` + `Transfer` dari 0x0).

## Gotcha VM
- Node via nvm: jalankan `source ~/.nvm/nvm.sh` di setiap shell baru.
- `pkill -f`/`pgrep -f` bisa ikut membunuh shell-nya sendiri. Pakai PID langsung.
- Gagal challenge **tidak** memotong kuota (`mintedByWallet` tetap) dan tidak ada biaya. Yang hilang cuma waktu.
