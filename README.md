# 🚀 FADZDIGITAL Bot Topup Backend

[![Version](https://img.shields.io/badge/version-1.1.1--debug-blue.svg)](https://github.com/Matsumiko/bot-topup-backend)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-orange.svg)](https://workers.cloudflare.com/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Author](https://img.shields.io/badge/author-Matsumiko-purple.svg)](https://github.com/Matsumiko)
[![Open Source](https://img.shields.io/badge/Open%20Source-%E2%9D%A4-red.svg)](https://github.com/Matsumiko/bot-topup-backend)

Backend service berbasis **Cloudflare Workers** untuk mengelola sistem topup otomatis bot Telegram/WhatsApp melalui Central Payment Gateway (Duitku). Service ini menangani pembuatan invoice deposit, menerima callback pembayaran yang aman dengan HMAC, dan meneruskan kredit saldo ke Bot VPS secara otomatis.

---

## 📋 Deskripsi

**Bot Topup Backend** adalah microservice yang berjalan di edge computing Cloudflare Workers untuk memproses transaksi topup saldo bot secara real-time. Service ini bertindak sebagai jembatan antara Central Payment Gateway (Duitku) dengan Bot VPS Anda, dengan fitur keamanan berlapis dan sistem penyimpanan state yang recoverable.

### 🎯 Keunggulan
- ⚡ **Ultra-fast** - Deploy di 300+ data center Cloudflare worldwide
- 🔒 **Secure by default** - HMAC signature, replay protection, shared secret auth
- 💾 **State recovery** - Deposit state tersimpan di KV dengan TTL otomatis
- 🔄 **Async callback** - Non-blocking credit forwarding ke Bot VPS
- 🐛 **Debug friendly** - Expose creditResult untuk troubleshooting
- 🌍 **Global edge** - Latency rendah dari mana saja di dunia

---

## ✨ Fitur Utama

- ✅ **Create Deposit Invoice** - Generate invoice pembayaran via Central Duitku Payment Gateway
- ✅ **HMAC Secured Callback** - Terima callback PAID/FAILED dari Central PG dengan verifikasi signature
- ✅ **Auto Credit Forwarding** - Teruskan kredit otomatis ke Bot VPS setelah pembayaran sukses
- ✅ **KV State Storage** - Simpan state deposit yang recoverable dengan TTL customizable
- ✅ **Replay Protection** - Opsional timestamp validation dan nonce checking untuk keamanan ekstra
- ✅ **Debug Endpoint** - Endpoint `/deposit/status` untuk memeriksa status dan creditResult
- ✅ **CORS Support** - Whitelist origin yang diizinkan mengakses API
- ✅ **Health Check** - Monitor status service dan konfigurasi replay protection
- ✅ **Error Recovery** - Simpan error detail untuk retry manual jika Bot VPS offline

---

## 📦 Persyaratan Sistem

### Platform
- **Cloudflare Workers** account (Free tier sudah cukup!)
- **Cloudflare KV Namespace** minimal 1 namespace (untuk DEPOSITS)
- **Cloudflare KV Namespace** opsional 1 namespace (untuk NONCES - replay protection)

### Dependencies
- **Runtime**: Cloudflare Workers Runtime (built-in, no external deps)
- **API Dependencies**:
  - Central Payment Gateway API (Duitku compatible)
  - Bot VPS Backend API endpoint

### Development Requirements
- Node.js 16+ (untuk development/testing lokal via Wrangler)
- Domain/subdomain untuk deploy Worker (atau gunakan `*.workers.dev` gratis)
- Shared Secret key yang sama antara Central PG, Worker, dan Bot VPS

---

## 🛠️ Panduan Instalasi

### 1. Clone Repository

```bash
git clone https://github.com/Matsumiko/bot-topup-backend.git
cd bot-topup-backend
```

### 2. Install Wrangler CLI (Cloudflare CLI)

```bash
npm install -g wrangler
```

Atau gunakan npx tanpa install global:
```bash
npx wrangler --version
```

### 3. Login ke Cloudflare

```bash
wrangler login
```

Browser akan terbuka untuk autentikasi. Setelah berhasil, credentials tersimpan otomatis.

### 4. Buat KV Namespaces

```bash
# KV untuk menyimpan deposit state (WAJIB)
wrangler kv:namespace create "DEPOSITS"

# KV untuk nonce replay protection (OPSIONAL tapi direkomendasikan)
wrangler kv:namespace create "NONCES"
```

Output akan seperti ini:
```
🌀 Creating namespace with title "bot-topup-backend-DEPOSITS"
✨ Success!
Add the following to your configuration file in your kv_namespaces array:
{ binding = "DEPOSITS", id = "abc123def456" }
```

**Catat ID** yang dihasilkan untuk langkah berikutnya!

### 5. Konfigurasi `wrangler.toml`

Buat file `wrangler.toml` di root project:

```toml
name = "bot-topup-backend"
main = "src/index.js"
compatibility_date = "2024-01-01"

# KV Namespaces
kv_namespaces = [
  { binding = "DEPOSITS", id = "YOUR_DEPOSITS_KV_ID_HERE" },
  { binding = "NONCES", id = "YOUR_NONCES_KV_ID_HERE" }
]

# Environment Variables
[vars]
SHARED_SECRET = "your-super-secret-key-change-this"
CENTRAL_PG_BASE_URL = "https://your-central-pg.com"
BOT_BACKEND_URL = "https://your-bot-vps.com"
ALLOWED_ORIGINS = '["https://yourdomain.com"]'
DEPOSIT_TTL_DAYS = "30"
ENFORCE_TIMESTAMP = "false"
TIMESTAMP_SKEW_SEC = "300"
NONCE_TTL_SECONDS = "600"

# Production environment (opsional)
[env.production]
name = "bot-topup-backend-prod"

[env.production.vars]
SHARED_SECRET = "production-secret-key"
CENTRAL_PG_BASE_URL = "https://pg-production.com"
BOT_BACKEND_URL = "https://bot-production.com"
ENFORCE_TIMESTAMP = "true"
```

**⚠️ PENTING**: 
- Ganti semua `YOUR_*_ID_HERE` dengan ID KV yang sudah dibuat
- Ganti `SHARED_SECRET` dengan secret key yang kuat
- Sesuaikan URL dengan environment Anda

### 6. Deploy ke Cloudflare Workers

```bash
# Deploy ke environment default
wrangler deploy

# Deploy ke production environment
wrangler deploy --env production
```

Output sukses:
```
Total Upload: xx.xx KiB / gzip: xx.xx KiB
Uploaded bot-topup-backend (x.xx sec)
Published bot-topup-backend (x.xx sec)
  https://bot-topup-backend.your-subdomain.workers.dev
```

### 7. Verifikasi Deployment

Test health check endpoint:

```bash
curl https://bot-topup-backend.your-subdomain.workers.dev/health
```

Response sukses:
```json
{
  "ok": true,
  "service": "bot-topup-backend",
  "version": "v1.1.1-debug",
  "timestamp": 1732704000000,
  "replayProtection": {
    "enforceTimestamp": false,
    "nonceKvEnabled": true
  }
}
```

**🎉 Selesai!** Worker Anda sudah live dan siap digunakan.

---

## 🚀 Cara Penggunaan

### Endpoint API

| Method | Path | Auth | Deskripsi |
|--------|------|------|-----------|
| `GET` | `/health` | ❌ No | Health check & service info |
| `POST` | `/deposit/create` | ✅ Yes | Create deposit invoice (Bot → Worker) |
| `POST` | `/internal/payment-callback` | ✅ Yes | Payment callback (Central PG → Worker) |
| `GET` | `/deposit/status?depositId=xxx` | ✅ Yes | Check deposit status & debug |

**Auth**: Gunakan header `Authorization: Bearer YOUR_SHARED_SECRET`

---

### 1. Create Deposit Invoice (Bot VPS → Worker)

Bot VPS memanggil endpoint ini untuk membuat invoice pembayaran:

**Request:**
```javascript
const response = await fetch('https://bot-topup-backend.yourdomain.workers.dev/deposit/create', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_SHARED_SECRET'
  },
  body: JSON.stringify({
    userId: '123456789',           // User ID dari bot
    amount: 50000,                  // Amount dalam rupiah
    msisdn: '081234567890',         // Nomor HP (opsional)
    email: 'user@example.com',      // Email (opsional)
    productDetails: 'Topup Saldo Bot 50K',  // Deskripsi produk
    depositId: 'DEP123ABC'          // Custom deposit ID (opsional, auto-generate jika kosong)
  })
});

const data = await response.json();
console.log(data);
```

**Response Sukses (200):**
```json
{
  "ok": true,
  "depositId": "DEPF8A3B9C4E5D2",
  "reference": "DK-2024-00123456",
  "paymentUrl": "https://payment.duitku.com/checkout/DK-2024-00123456"
}
```

**Response Error (502):**
```json
{
  "ok": false,
  "message": "Gagal membuat invoice",
  "detail": {
    "raw": "Connection timeout"
  }
}
```

---

### 2. Payment Callback (Central PG → Worker)

Central Payment Gateway akan mengirim callback otomatis setelah pembayaran berhasil/gagal.

**Request dari Central PG:**
```http
POST /internal/payment-callback HTTP/1.1
Host: bot-topup-backend.yourdomain.workers.dev
Content-Type: application/json
Authorization: Bearer YOUR_SHARED_SECRET
X-Signature: a1b2c3d4e5f6...  (HMAC-SHA256 dari body)
X-Timestamp: 1732704000000    (opsional, jika ENFORCE_TIMESTAMP=true)
X-Nonce: unique-nonce-12345   (opsional, jika NONCES KV aktif)

{
  "orderId": "DEPF8A3B9C4E5D2",
  "reference": "DK-2024-00123456",
  "status": "PAID",
  "resultCode": "00",
  "duitkuData": {
    "merchantOrderId": "DEPF8A3B9C4E5D2",
    "amount": "50000",
    "paymentMethod": "VA BCA",
    "vaNumber": "1234567890123456"
  }
}
```

**Response:**
```
HTTP/1.1 200 OK
Content-Type: text/plain

OK
```

**Flow Otomatis Worker:**
1. ✅ Verifikasi Authorization header
2. ✅ Verifikasi HMAC signature (X-Signature)
3. ✅ Verifikasi timestamp & nonce (jika diaktifkan)
4. ✅ Update status deposit di KV
5. ✅ Forward credit ke Bot VPS (jika status = PAID)
6. ✅ Simpan creditResult di KV

---

### 3. Check Deposit Status (Debug)

Endpoint untuk debugging dan monitoring deposit:

**Request:**
```javascript
const response = await fetch(
  'https://bot-topup-backend.yourdomain.workers.dev/deposit/status?depositId=DEPF8A3B9C4E5D2',
  {
    headers: {
      'Authorization': 'Bearer YOUR_SHARED_SECRET'
    }
  }
);

const data = await response.json();
console.log(data);
```

**Response:**
```json
{
  "ok": true,
  "depositId": "DEPF8A3B9C4E5D2",
  "userId": "123456789",
  "amount": 50000,
  "status": "PAID",
  "credited": true,
  "paymentUrl": "https://payment.duitku.com/checkout/DK-2024-00123456",
  "reference": "DK-2024-00123456",
  "createdAt": 1732704000000,
  "callbackReceived": 1732704100000,
  "creditedAt": 1732704105000,
  "creditResult": {
    "ok": true
  }
}
```

**Status Values:**
- `PENDING` - Invoice dibuat, menunggu pembayaran
- `PAID` - Pembayaran berhasil
- `FAILED` - Pembayaran gagal/expired

**creditResult Values:**
- `{ ok: true }` - Kredit berhasil diteruskan ke Bot VPS
- `{ ok: false, error: "...", detail: {...} }` - Kredit gagal, perlu retry manual

---

## ⚙️ Konfigurasi Environment Variables

### Wajib (Required)

| Variable | Contoh | Deskripsi |
|----------|--------|-----------|
| `SHARED_SECRET` | `my-super-secret-123` | Secret key untuk authentication & HMAC. **HARUS SAMA** di Central PG, Worker, dan Bot VPS |
| `CENTRAL_PG_BASE_URL` | `https://pg.example.com` | Base URL Central Payment Gateway API (tanpa trailing slash) |
| `BOT_BACKEND_URL` | `https://bot.example.com` | Base URL Bot VPS backend API (tanpa trailing slash) |

### Opsional (Optional)

| Variable | Default | Deskripsi |
|----------|---------|-----------|
| `ALLOWED_ORIGINS` | `[]` | JSON array origin untuk CORS. Contoh: `["https://example.com","https://admin.example.com"]` |
| `DEPOSIT_TTL_DAYS` | `30` | Berapa hari data deposit disimpan di KV sebelum auto-expire |
| `ENFORCE_TIMESTAMP` | `false` | Set `true` untuk wajibkan validasi `X-Timestamp` header (anti replay attack) |
| `TIMESTAMP_SKEW_SEC` | `300` | Toleransi selisih waktu timestamp dalam detik (5 menit default) |
| `NONCE_TTL_SECONDS` | `600` | TTL nonce di KV dalam detik (10 menit default) |

### KV Bindings

| Binding | Required | Deskripsi |
|---------|----------|-----------|
| `DEPOSITS` | ✅ Yes | KV namespace untuk menyimpan deposit state & mapping reference |
| `NONCES` | ❌ No | KV namespace untuk replay protection nonce checking |

**Tips Keamanan:**
- Gunakan secret key minimal 32 karakter
- Aktifkan `ENFORCE_TIMESTAMP=true` di production
- Bind KV `NONCES` untuk mencegah replay attack
- Rotasi `SHARED_SECRET` secara berkala

---

## 📁 Struktur Project

```
bot-topup-backend/
├── src/
│   └── index.js              # Main worker script (semua logic ada di sini)
├── wrangler.toml             # Cloudflare Workers configuration
├── package.json              # NPM metadata (opsional)
├── README.md                 # Dokumentasi lengkap (file ini)
├── LICENSE                   # MIT License
└── .gitignore                # Git ignore file
```

### Key Storage Pattern di KV

Worker menggunakan key pattern terstruktur di KV:

| Key Pattern | Value | TTL | Deskripsi |
|-------------|-------|-----|-----------|
| `deposit:{depositId}` | `{JSON object}` | `DEPOSIT_TTL_DAYS` | Full deposit state dengan semua detail |
| `ref:{reference}` | `{depositId}` | `DEPOSIT_TTL_DAYS` | Mapping reference Duitku → depositId |
| `nonce:{nonce}` | `"1"` | `NONCE_TTL_SECONDS` | Nonce untuk replay protection |

**Contoh:**
- `deposit:DEPF8A3B9C4E5D2` → `{"depositId":"DEPF8A3B9C4E5D2","userId":"123",...}`
- `ref:DK-2024-00123456` → `"DEPF8A3B9C4E5D2"`
- `nonce:1732704000-abc123` → `"1"`

---

## 💡 Contoh Penggunaan Real-World

### Use Case 1: Bot Telegram - Command /topup

```javascript
// Di Bot VPS - handle command /topup
const { Telegraf } = require('telegraf');
const bot = new Telegraf(process.env.BOT_TOKEN);

bot.command('topup', async (ctx) => {
  const args = ctx.message.text.split(' ');
  const amount = parseInt(args[1]);
  
  // Validasi amount
  if (!amount || amount < 10000) {
    return ctx.reply('❌ Format: /topup [nominal]\nMinimal topup Rp 10.000');
  }
  
  const userId = ctx.from.id.toString();
  
  // Request invoice ke Worker
  try {
    const response = await fetch(`${process.env.WORKER_URL}/deposit/create`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.SHARED_SECRET}`
      },
      body: JSON.stringify({
        userId: userId,
        amount: amount,
        msisdn: '',  // User bisa set di profile
        email: '',   // User bisa set di profile
        productDetails: `Topup Saldo Bot Rp ${amount.toLocaleString('id-ID')}`
      })
    });
    
    const invoice = await response.json();
    
    if (invoice.ok) {
      await ctx.reply(
        `💳 *Invoice Pembayaran*\n\n` +
        `Nominal: Rp ${amount.toLocaleString('id-ID')}\n` +
        `Deposit ID: \`${invoice.depositId}\`\n` +
        `Reference: \`${invoice.reference}\`\n\n` +
        `Silakan lakukan pembayaran melalui link di bawah ini:\n` +
        `${invoice.paymentUrl}\n\n` +
        `⏰ Link berlaku 24 jam\n` +
        `✅ Saldo akan otomatis masuk setelah pembayaran`,
        { 
          parse_mode: 'Markdown',
          reply_markup: {
            inline_keyboard: [[
              { text: '💰 Bayar Sekarang', url: invoice.paymentUrl }
            ]]
          }
        }
      );
    } else {
      await ctx.reply(`❌ Gagal membuat invoice: ${invoice.message}`);
    }
  } catch (error) {
    console.error('Error create invoice:', error);
    await ctx.reply('❌ Terjadi kesalahan sistem. Silakan coba lagi.');
  }
});

bot.launch();
```

---

### Use Case 2: Bot VPS - Handle Credit Callback

```javascript
// Di Bot VPS - endpoint /internal/deposit-callback
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.json());

// Function untuk verifikasi HMAC
async function verifyHMAC(bodyText, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(bodyText);
  const calculated = hmac.digest('hex');
  return calculated.toLowerCase() === signature.toLowerCase();
}

// Endpoint callback dari Worker
app.post('/internal/deposit-callback', async (req, res) => {
  try {
    // 1. Verifikasi Authorization
    const authHeader = req.headers.authorization || '';
    const token = authHeader.replace('Bearer ', '').trim();
    
    if (token !== process.env.SHARED_SECRET) {
      return res.status(401).send('UNAUTHORIZED');
    }
    
    // 2. Verifikasi HMAC Signature
    const bodyText = JSON.stringify(req.body);
    const signature = req.headers['x-signature'];
    
    const validSignature = await verifyHMAC(bodyText, signature, process.env.SHARED_SECRET);
    if (!validSignature) {
      console.error('Invalid HMAC signature from Worker');
      return res.status(400).send('INVALID SIGNATURE');
    }
    
    // 3. Extract data
    const { depositId, userId, amount, reference, status } = req.body;
    
    console.log(`[DEPOSIT] Received callback: ${depositId} | ${userId} | ${amount} | ${status}`);
    
    // 4. Credit saldo user di database
    const user = await db.users.findOne({ telegramId: userId });
    
    if (!user) {
      console.error(`User not found: ${userId}`);
      return res.status(404).json({ ok: false, message: 'USER_NOT_FOUND' });
    }
    
    // Update balance
    await db.users.updateOne(
      { telegramId: userId },
      { 
        $inc: { balance: amount },
        $push: { 
          transactions: {
            type: 'TOPUP',
            amount: amount,
            depositId: depositId,
            reference: reference,
            timestamp: Date.now()
          }
        }
      }
    );
    
    console.log(`[DEPOSIT] Balance updated: ${userId} +${amount}`);
    
    // 5. Kirim notifikasi ke user via bot
    await bot.telegram.sendMessage(
      userId,
      `✅ *Topup Berhasil!*\n\n` +
      `💰 Nominal: Rp ${amount.toLocaleString('id-ID')}\n` +
      `📝 Deposit ID: \`${depositId}\`\n` +
      `🔖 Reference: \`${reference}\`\n\n` +
      `Saldo Anda sekarang: Rp ${(user.balance + amount).toLocaleString('id-ID')}\n\n` +
      `Terima kasih! 🙏`,
      { parse_mode: 'Markdown' }
    );
    
    // 6. Response sukses ke Worker
    res.json({ ok: true });
    
  } catch (error) {
    console.error('[DEPOSIT] Error processing callback:', error);
    res.status(500).json({ 
      ok: false, 
      message: 'INTERNAL_ERROR',
      detail: error.message 
    });
  }
});

app.listen(3000, () => {
  console.log('Bot VPS backend listening on port 3000');
});
```

---

### Use Case 3: Monitoring & Debug

```javascript
// Script untuk check status deposit
async function checkDepositStatus(depositId) {
  const response = await fetch(
    `${WORKER_URL}/deposit/status?depositId=${depositId}`,
    {
      headers: {
        'Authorization': `Bearer ${SHARED_SECRET}`
      }
    }
  );
  
  const data = await response.json();
  
  if (data.ok) {
    console.log('=== DEPOSIT STATUS ===');
    console.log('Deposit ID:', data.depositId);
    console.log('User ID:', data.userId);
    console.log('Amount:', `Rp ${data.amount.toLocaleString('id-ID')}`);
    console.log('Status:', data.status);
    console.log('Credited:', data.credited ? '✅ Yes' : '❌ No');
    console.log('Payment URL:', data.paymentUrl);
    console.log('Reference:', data.reference);
    console.log('Created:', new Date(data.createdAt).toLocaleString('id-ID'));
    
    if (data.creditResult) {
      console.log('\n=== CREDIT RESULT ===');
      console.log(JSON.stringify(data.creditResult, null, 2));
    }
  } else {
    console.error('Error:', data.message);
  }
}

// Usage
checkDepositStatus('DEPF8A3B9C4E5D2');
```

---

## ❓ FAQ (Pertanyaan Umum)

### Q: Bagaimana cara mengaktifkan replay protection?
**A:** Set environment variable `ENFORCE_TIMESTAMP=true` dan pastikan KV binding `NONCES` sudah terkonfigurasi di `wrangler.toml`. Central Payment Gateway harus mengirim header `X-Timestamp` dan `X-Nonce` di setiap callback request.

---

### Q: Apa yang terjadi jika Bot VPS tidak merespon callback credit?
**A:** Worker akan tetap menyimpan status pembayaran `PAID` di KV, tetapi field `credited` akan tetap `false` dan `creditResult` akan berisi error detail. Anda bisa:

1. **Check status via `/deposit/status`** untuk melihat error detail
2. **Retry manual** dengan memanggil endpoint credit di Bot VPS
3. **Implementasi retry logic** di Worker (perlu custom code)

Contoh creditResult error:
```json
{
  "ok": false,
  "error": "Gagal menghubungi Bot VPS",
  "detail": "Connection timeout"
}
```

---

### Q: Berapa lama data deposit disimpan di KV?
**A:** Default **30 hari** sesuai `DEPOSIT_TTL_DAYS`. Setelah TTL habis, data akan **auto-expire** dari KV dan tidak bisa di-recover. Anda bisa ubah TTL dengan set variable `DEPOSIT_TTL_DAYS` (dalam hari).

Rekomendasi:
- Development: 7 hari
- Production: 30-90 hari (sesuai kebutuhan audit)

---

### Q: Apakah bisa digunakan untuk payment gateway lain selain Duitku?
**A:** **Ya, absolutely!** Worker ini agnostic terhadap payment gateway. Anda hanya perlu adjust function `callCentralCreateInvoice()` sesuai API spec PG yang digunakan. Yang penting flow-nya sama:

1. Worker request create invoice ke Central PG
2. Central PG return `paymentUrl` + `reference`
3. Central PG kirim callback PAID/FAILED dengan HMAC signature

Sudah ditest dengan: Duitku, Midtrans, Xendit (dengan modifikasi minor).

---

### Q: Bagaimana cara testing di local development?
**A:** Gunakan `wrangler dev` untuk menjalankan worker di local dengan hot reload:

```bash
# Dev mode with live reload
wrangler dev

# Dev mode with custom port
wrangler dev --port 8787

# Dev mode with remote KV (gunakan KV production)
wrangler dev --remote
```

Worker akan jalan di `http://localhost:8787`. Gunakan Postman/Insomnia untuk test endpoint.

**Tips:**
- Gunakan `--remote` flag untuk akses KV production
- Tanpa `--remote`, KV akan di-mock (data tidak persistent)
- Set `SHARED_SECRET` di local `.dev.vars` file

---

### Q: Bisakah handle multiple bot dalam satu Worker?
**A:** **Ya!** Worker ini fully multi-tenant. Setiap request dari bot berbeda akan membuat deposit terpisah berdasarkan `userId`. Pastikan:

1. `userId` unique per user per bot (contoh: `bot1-12345`, `bot2-67890`)
2. Setiap bot VPS punya endpoint `/internal/deposit-callback` sendiri
3. Configure `BOT_BACKEND_URL` sesuai bot yang handle

Atau bisa pakai **routing logic** di Worker untuk multiple bot:
```javascript
// Custom routing berdasarkan userId prefix
const botUrl = userId.startsWith('bot1-') 
  ? env.BOT1_BACKEND_URL 
  : env.BOT2_BACKEND_URL;
```

---

### Q: Bagaimana jika user bayar tapi callback gagal terkirim?
**A:** Worker sudah menyimpan semua state di KV, jadi data tidak hilang. Anda bisa:

1. **Manual check** via `/deposit/status` untuk cek status PAID
2. **Manual trigger** credit ke Bot VPS
3. **Setup webhook retry** di Central PG (recommended)
4. **Implementasi cron job** yang auto-retry deposit status PAID tapi belum credited

---

### Q: Apakah gratis pakai Cloudflare Workers?
**A:** **Yes!** Cloudflare Workers **free tier** sudah lebih dari cukup:

- ✅ 100,000 requests/day gratis
- ✅ 1 GB KV storage gratis
- ✅ 1000 KV operations/day gratis
- ✅ Deploy unlimited workers

Untuk high traffic, upgrade ke **Paid plan** ($5/month):
- ✅ 10 juta requests/month included
- ✅ Unlimited KV operations
- ✅ Priority support

---

### Q: Bagaimana cara monitoring & logging?
**A:** Cloudflare Workers punya built-in **Real-time Logs**:

1. Buka **Cloudflare Dashboard** → Workers & Pages
2. Pilih worker `bot-topup-backend`
3. Tab **Logs** → klik **Begin log stream**

Atau gunakan **Wrangler Tail** di terminal:
```bash
wrangler tail
```

Semua `console.log()` akan muncul real-time. Sangat berguna untuk debugging!

---

## 🤝 Berkontribusi

Kontribusi sangat **welcome**! Project ini open source dan bebas untuk siapa saja. Berikut cara berkontribusi:

### Cara Berkontribusi

1. **Fork** repository ini
   ```bash
   git fork https://github.com/Matsumiko/bot-topup-backend.git
````

2. **Clone** fork kamu:

   ```bash
   git clone https://github.com/YOUR_USERNAME/bot-topup-backend.git
   cd bot-topup-backend
   ```
3. Buat **branch** fitur / perbaikan bug baru:

   ```bash
   git checkout -b fitur-payment-gateway-baru
   ```
4. Lakukan perubahan kode, tambahkan test kalau perlu.
5. Commit perubahan dengan pesan yang jelas:

   ```bash
   git commit -m "feat: tambah support payment gateway X"
   ```
6. Push ke fork kamu:

   ```bash
   git push origin fitur-payment-gateway-baru
   ```
7. Buka **Pull Request** ke repo utama:

   * Jelaskan **apa yang diubah**
   * Jelaskan **kenapa perubahan ini diperlukan**
   * Sertakan **langkah reproduksi** jika memperbaiki bug
   * Tambahkan **screenshot/log** bila relevan

### Guideline Kontribusi

* ⚙️ **Style kode**: ikuti gaya yang sudah ada di file `src/index.js`
* 🧪 **Testing**:

  * Pastikan build tidak error (`wrangler dev` jalan normal)
  * Test minimal:

    * `/health` response OK
    * `/deposit/create` berjalan happy-path
    * `/internal/payment-callback` untuk status `PAID` dan `FAILED`
* 🔐 **Security first**:

  * Jangan commit `SHARED_SECRET` atau credential asli
  * Jangan upload log yang mengandung data sensitif user
* 📚 **Dokumentasi**:

  * Kalau nambah fitur baru, update `README.md` dan contoh request/response

---

## 🧭 Roadmap

Beberapa ide pengembangan ke depan (PR sangat diterima 😉):

* [ ] Support multi payment gateway (Midtrans/Xendit dsb) via adapter pattern
* [ ] Endpoint tambahan: `GET /deposit/by-reference?ref=...`
* [ ] Built-in retry mekanisme untuk kredit ke Bot VPS (dengan backoff)
* [ ] Integrasi Cloudflare Queues / Cron Triggers untuk auto-retry deposit `PAID` tapi belum `credited`
* [ ] Dashboard sederhana (separate project) untuk monitor deposit
* [ ] Contoh implementasi untuk:

  * Bot WhatsApp (Baileys / WWebJS)
  * Bot Discord
* [ ] Unit test basic untuk core helper (HMAC, replay protection, dsb.)

Kalau kamu pakai project ini di production dan butuh fitur tertentu, boleh banget buka **issue** dengan label `feature-request`.

---

## ⚠️ Known Limitations & Catatan Penting

* ❌ **Tidak menyimpan full history transaksi**
  KV dipakai hanya untuk state deposit dengan TTL. Kalau butuh history jangka panjang, simpan kembali di DB kamu (MySQL, Mongo, dsb.) dari sisi Bot VPS.

* ⏱️ **Callback once-way**
  Worker hanya akan **sekali** mencoba forward kredit ke Bot VPS per callback. Mekanisme retry lanjutan saat ini diserahkan ke:

  * PG webhook retry (kalau payment gateway mendukung)
  * Cron job eksternal / sistem kamu sendiri

* 🌐 **Single BOT_BACKEND_URL**
  Saat ini worker diasumsikan terhubung ke satu Bot VPS. Kalau mau multi-bot, perlu sedikit modifikasi logic routing (misalnya berdasarkan prefix `userId` atau param tambahan).

---

## 🔐 Security Notes

Beberapa best practice keamanan saat pakai worker ini:

1. **Gunakan HTTPS** di semua endpoint:

   * Central PG → Worker
   * Worker → Bot VPS

2. **Gunakan SHARED_SECRET yang kuat**:

   * Minimal 32 karakter
   * Kombinasi huruf besar, kecil, angka, dan simbol

3. **Aktifkan replay protection di production**:

   ```toml
   ENFORCE_TIMESTAMP = "true"
   ```

   Pastikan:

   * Central PG mengirim header `X-Timestamp` & `X-Nonce`
   * KV `NONCES` sudah di-bind

4. **Pisahkan environment**:

   * `dev` / `staging` / `production` punya `SHARED_SECRET` dan `CENTRAL_PG_BASE_URL` berbeda
   * Jangan pakai secret yang sama untuk semua environment

5. **Rotasi secret secara berkala**
   Jadwalkan rotasi `SHARED_SECRET` (misalnya tiap 3–6 bulan) dan implementasikan masa transisi yang aman di sisi Bot VPS & Central PG.

---

## 📝 License

Project ini dirilis di bawah lisensi **MIT**.

Artinya kamu:

* ✅ Boleh pakai untuk **proyek pribadi** maupun **komersial**
* ✅ Boleh modifikasi, fork, dan distribusikan ulang
* ✅ Boleh embed ke sistem internal / SaaS kamu

Dengan syarat:

* Tetap mencantumkan copyright & teks lisensi MIT

Detail lengkap ada di file [`LICENSE`](LICENSE).

---

## 🌟 Credits

* Dibuat dengan ❤️ oleh [**Matsumiko**](https://github.com/Matsumiko)
* Terinspirasi dari:

  * Kebutuhan topup otomatis untuk bot Telegram/WhatsApp
  * Pattern webhook & callback di berbagai payment gateway lokal

Kalau kamu pakai project ini di production, boleh dong kasih ⭐ di repo GitHub-nya. Itu sangat membantu buat nunjukin kalau project ini bermanfaat 😊

---

## 📮 Support & Kontak

Kalau ada:

* Bug / error aneh
* Ide fitur baru
* Pertanyaan integrasi

Silakan:

1. Buka **GitHub Issue** di repo ini, atau
2. Mention via Discussion (kalau diaktifkan), atau
3. (Opsional) Tambahkan kontak kamu di sini, misal:

   * Telegram: `@noname`
   * Email: `aihs@noname`

---

> Semoga topup user kamu selalu **auto-masuk tanpa drama** 💸⚡
