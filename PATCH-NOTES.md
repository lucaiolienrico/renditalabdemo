# RenditaLab Demo — Audit & Patch

Repo: `lucaiolienrico/renditabdemo` · Backend verificato live via Supabase MCP.

## Verdetto per layer

| Layer | Stato | Note |
|---|---|---|
| Supabase (DB) | OK | RLS attivo; `trials` PK user_id + FK auth.users CASCADE; policy `SELECT own`; nessuna write client (solo RPC/service_role). `paid_emails` PK email, RLS lockdown (0 policy = inaccessibile ai client). |
| Supabase (RPC) | OK | `ensure_trial` SECURITY DEFINER + `search_path=public` + guard `auth.uid()`; trial default `now()+3 days`; `is_valid = paid OR expires_at>now()`. |
| Edge Functions | OK | `paypal-create-order` (amount+custom_id server-side), `paypal-capture-order` (valida COMPLETED + custom_id==user + ccy + amount≥49), `paypal-webhook` (verifica firma, match UUID, fallback email). JWT enforce corretto. |
| PayPal | OK (1 nota) | Flusso Orders API robusto. Vedi #C4. |
| index.html | CORRETTO | Vedi #C1–#C3. |
| GitHub repo | CORRETTO | Vedi #C1, #C5. |
| Vercel | AGGIUNTO | `vercel.json` assente → creato (security headers). |

## Criticità trovate e correzioni

| # | Sev | Problema | Fix |
|---|---|---|---|
| C1 | ALTA | URL self-referential rotti: `canonical`/`og:url`/`og:image`/schema puntavano a `lucaiolienrico.github.io/renditalabdemo/`. Path GH Pages di QUESTO repo = `/rendimaxdemo/`, non `/renditalabdemo/` → 404. README "Live" indica invece Vercel. | Allineati i 6 self-URL a `https://renditalabdemo.vercel.app/` (host con prova "Live"). |
| C2 | MEDIA | README brand "Rendimax" (vietato in output pubblico) + contraddizione "usa GitHub Pages" vs Live su `vercel.app`. | README riscritto: brand RenditaLab, host Vercel, URL coerenti. |
| C3 | MEDIA | Assenti `robots.txt`, `sitemap.xml`, `llms.txt`. Pagina `index,follow` senza direttive crawler/GEO. | Creati i 3 file, host-coerenti. |
| C4 | MEDIA | `vercel.json` assente → nessun header HTTP reale; sicurezza affidata solo al meta-CSP (no HSTS, no X-Frame-Options, no frame-ancestors). | `vercel.json` con CSP (mirror del meta + `frame-ancestors 'none'`), HSTS, X-Content-Type-Options, X-Frame-Options DENY, Referrer-Policy, Permissions-Policy. |
| C5 | BASSA | Nome repo `rendimaxdemo` = brand legacy. | Vedi "Decisioni aperte". |

## Non-bug (verificati, lasciati invariati)

- **Anon key in chiaro** (`index.html`): è la chiave pubblica `anon` (JWT ref `xpnpdkaaeudufupovsxm`, role anon, exp 2036). Esposizione corretta by-design.
- **Gating risultati = blur CSS** (aggirabile via DevTools): già documentato nel codice. Accettabile per gate lead-gen; il trial/pagamento è enforced server-side. Per protezione reale → spostare il calcolo in Edge Function.
- **Supabase project ≠ canonical RenditaLab** (`amzjefyegfxkpzjifynj`): la demo usa progetto dedicato `xpnpdkaaeudufupovsxm`. Isolamento sensato per una demo → lasciato. ⚠️ Conferma che sia intenzionale.
- **Advisor WARN** `ensure_trial` eseguibile da `authenticated`: intenzionale (la usa `check-trial` col JWT utente). Funzione sicura.
- **Advisor INFO** `paid_emails` RLS senza policy: intenzionale (lockdown).

## Note minori (no fix automatico)

- `paypal-webhook` include `CHECKOUT.ORDER.APPROVED` tra i PAID_EVENTS. APPROVED ≠ catturato: rischio teorico di sblocco pre-capture. Rischio basso (guard importo presente). Valutare di tenere solo `PAYMENT.CAPTURE.COMPLETED`.
- CORS `Access-Control-Allow-Origin: *` su tutte le function. Le 3 con JWT sono protette comunque; valutare allowlist dell'origin demo.

## Decisioni aperte (richiedono tua conferma)

1. **Host definitivo**: ho assunto **Vercel** (`renditalabdemo.vercel.app`). Se invece è GitHub Pages → rinomina il repo in `renditalabdemo` e sostituisci i self-URL con `https://lucaiolienrico.github.io/renditalabdemo/` (e ignora `vercel.json`).
2. **SEO canonical → cannibalizzazione**: la demo compete con `renditalab.it` sulle stesse keyword. Opzione: settare `canonical` della demo verso la relativa pagina di `renditalab.it` per consolidare autorità (la demo non si posizionerebbe da sola).
3. **Rinomina repo** `rendimaxdemo` → `renditalabdemo` (coerenza brand; rompe URL GH Pages se attivi).

## Deploy manuale (no push automatico)

1. Copia i file in `out/` nella root del repo locale.
2. GitHub Desktop → commit → push su `main`.
3. Vercel: verifica che `vercel.json` sia rilevato (Deployments → Headers).
4. Search Console: invia `https://renditalabdemo.vercel.app/sitemap.xml`.
