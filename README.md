# RenditaLab — Demo pubblica

Demo del calcolatore di rendimento immobiliare **RenditaLab**: cash flow, rendimento netto, ROI, Cap Rate, simulazione mutuo e proiezione a 10 anni con fiscalità italiana (cedolare secca 21% / 10%, IRPEF).

🔗 **Live**: https://renditalabdemo.vercel.app/
🏠 **Prodotto completo**: https://renditalab.it

## Architettura

- `index.html` — demo standalone single-file, zero dipendenze runtime (jsPDF, Supabase JS e PayPal SDK caricati via CDN).
- Backend trial/pagamenti: progetto Supabase dedicato `xpnpdkaaeudufupovsxm` (Edge Functions: `check-trial`, `paypal-create-order`, `paypal-capture-order`, `paypal-webhook`).
- Pagamenti: PayPal Orders API (Smart Buttons) — importo e `custom_id` fissati server-side, anti-manomissione.

## Deploy

Host: **Vercel** (`renditalabdemo.vercel.app`). Deploy manuale tramite GitHub Desktop / dashboard Vercel.
Gli header di sicurezza HTTP sono definiti in `vercel.json`; il meta-CSP in `index.html` resta come fallback.

## File

| File | Ruolo |
|---|---|
| `index.html` | Demo + gate freemium + integrazione PayPal/Supabase |
| `vercel.json` | Security headers (CSP, HSTS, X-Frame-Options, ecc.) |
| `robots.txt` | Direttive crawler + riferimento sitemap |
| `sitemap.xml` | Sitemap (1 URL) |
| `llms.txt` | Descrizione per motori generativi (GEO) |
| `og-image.png` | Open Graph 1200×630 |
