# Spesifikasi REST API Veris (`v1`)

> **Base Path:** `/api/v1`  
> **Format Pertukaran Data:** `application/json` (UTF-8)  
> **Autentikasi Opsional:** `X-Veris-API-Key: <token>` atau `Authorization: Bearer <token>`

---

## 1. Ringkasan Endpoint Produksi

| Metode | Endpoint | Deskripsi | Ukuran Input Maks. |
| :---: | :--- | :--- | :---: |
| `GET` | `/api/v1/health` | Status kesehatan mesin, kesiapan model IndoBERT, telemetri RAM (`memory_rss_mb`), & verifikasi `0% Cloud Egress`. | — |
| `GET` | `/api/v1/taxonomy` | Daftar 10 entitas PII Indonesia beserta kategori UU PDP, referensi pasal, aturan validasi, & bobot risiko. | — |
| `POST` | `/api/v1/detect` | Deteksi span PII presisi tinggi beserta indeks karakter `(start, end)`, *confidence score*, sumber detektor, & skor risiko UU PDP. | `50.000` karakter |
| `POST` | `/api/v1/redact` | De-identifikasi teks dengan 4 pilihan strategi (`tag`, `mask`, `hash`, `surrogate`) beserta kamus `vault_mapping`. | `50.000` karakter |
| `POST` | `/api/v1/batch-redact` | De-identifikasi massal hingga `100` dokumen sekaligus dalam satu permintaan HTTP. | `100` dok (`20.000` kar/dok) |
| `POST` | `/api/v1/gateway` | Siklus penuh **Pre-LLM Privacy Gateway**: sanitasi keluar, *Prompt Injection Guardrail*, simulasi LLM, & re-identifikasi lokal. | `25.000` karakter |

---

## 2. Detail Skema & Contoh Payload

### A. `GET /api/v1/health`

**Response (`200 OK`):**
```json
{
  "status": "healthy",
  "app_name": "Veris PII Protection Engine",
  "version": "1.0.0",
  "deployment_mode": "ON_PREMISE_CPU",
  "indobert_loaded": true,
  "structural_validator_ready": true,
  "cpu_threads": 4,
  "memory_rss_mb": 540.2,
  "cloud_egress_percent": 0.0,
  "supported_entities": [
    "NIK", "NO_KK", "NPWP", "NO_BPJS", "NO_REKENING",
    "NAMA", "TANGGAL_LAHIR", "ALAMAT", "TELEPON", "EMAIL"
  ]
}
```

---

### B. `POST /api/v1/redact`

**Request Body:**
```json
{
  "text": "Mohon diproses restrukturisasi kredit a/n Hendra Wijaya Kusuma (NIK: 3273011508900003, NPWP: 09.234.567.8-412.000). Pencairan ke rekening BCA 1300098765432. Nomor tiket internal #TRX-8829104401928374.",
  "mode": "tag",
  "entities": null
}
```
- Parameter `mode` mendukung nilai: `"tag"` *(default)*, `"mask"`, `"hash"`, atau `"surrogate"`.

**Response (`200 OK`):**
```json
{
  "original_text": "Mohon diproses restrukturisasi kredit a/n Hendra Wijaya Kusuma (NIK: 3273011508900003, NPWP: 09.234.567.8-412.000). Pencairan ke rekening BCA 1300098765432. Nomor tiket internal #TRX-8829104401928374.",
  "redacted_text": "Mohon diproses restrukturisasi kredit a/n [NAMA_1] (NIK: [NIK_1], NPWP: [NPWP_1]). Pencairan ke rekening BCA [NO_REKENING_1]. Nomor tiket internal #TRX-8829104401928374.",
  "mode": "tag",
  "vault_mapping": {
    "[NAMA_1]": "Hendra Wijaya Kusuma",
    "[NIK_1]": "3273011508900003",
    "[NPWP_1]": "09.234.567.8-412.000",
    "[NO_REKENING_1]": "1300098765432"
  },
  "compliance": {
    "total_pii_spans": 4,
    "specific_pii_count": 3,
    "general_pii_count": 1,
    "validated_checksum_count": 3,
    "risk_score": 100.0,
    "risk_level": "CRITICAL",
    "counts_by_entity": {
      "NAMA": 1,
      "NIK": 1,
      "NPWP": 1,
      "NO_REKENING": 1
    },
    "cloud_egress_percent": 0.0
  },
  "latency_ms": 18.4,
  "engine": "Veris Hybrid Engine v1.0 (IndoBERT-21BIO + Statutory Checksum Validator)"
}
```

---

### C. `POST /api/v1/gateway` (Pre-LLM Privacy Gateway)

**Request Body:**
```json
{
  "prompt": "Berdasarkan berkas layanan publik, pemohon dr. Pratama Adi Nugroho (Lahir: 22 November 1986) dengan NIK 3171042211860002 dan Nomor KK 3171040905120004 mengajukan pembaruan JKN-BPJS 0001458923417.",
  "assistant_context": "govtech"
}
```

**Response (`200 OK`):**
```json
{
  "raw_user_prompt": "Berdasarkan berkas layanan publik, pemohon dr. Pratama Adi Nugroho (Lahir: 22 November 1986) dengan NIK 3171042211860002 dan Nomor KK 3171040905120004 mengajukan pembaruan JKN-BPJS 0001458923417.",
  "sanitized_outbound_prompt": "Berdasarkan berkas layanan publik, pemohon [NAMA_1] (Lahir: [TANGGAL_LAHIR_1]) dengan NIK [NIK_1] dan Nomor KK [NO_KK_1] mengajukan pembaruan JKN-BPJS [NO_BPJS_1].",
  "simulated_llm_response": "Yth. [NAMA_1] (Tanggal Lahir: [TANGGAL_LAHIR_1]), permohonan pembaruan data kependudukan untuk NIK [NIK_1] (Kartu Keluarga [NO_KK_1]) serta sinkronisasi kepesertaan JKN-BPJS [NO_BPJS_1] telah diverifikasi sesuai struktur regulasi layanan publik.",
  "reidentified_user_response": "Yth. dr. Pratama Adi Nugroho (Tanggal Lahir: 22 November 1986), permohonan pembaruan data kependudukan untuk NIK 3171042211860002 (Kartu Keluarga 3171040905120004) serta sinkronisasi kepesertaan JKN-BPJS 0001458923417 telah diverifikasi sesuai struktur regulasi layanan publik.",
  "vault_mapping": {
    "[NAMA_1]": "dr. Pratama Adi Nugroho",
    "[TANGGAL_LAHIR_1]": "22 November 1986",
    "[NIK_1]": "3171042211860002",
    "[NO_KK_1]": "3171040905120004",
    "[NO_BPJS_1]": "0001458923417"
  },
  "guardrail": {
    "blocked": false,
    "threat_type": null,
    "matched_pattern": null,
    "sanitized_spoofed_tags": 0,
    "policy": "OWASP-LLM01 / MITRE-ATLAS-AML.T0051 Active"
  },
  "latency_ms": 21.8,
  "zero_cloud_pii_leakage": true
}
```
