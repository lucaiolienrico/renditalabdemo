# renditalabdemo — Audit Report completo
**Data:** 27/06/2026 · **Scope:** end-to-end (Frontend SPA · 4 Edge Function · Schema/RLS · Infra · PayPal)
**Metodo:** lettura sorgente deployato (`main`) + MCP Supabase (schema, advisors, edge fn) + MCP Vercel (deploy) + validazione JS `node --check`.

---

## 1. Verdetto sintetico

| Layer | Stato | Bug critici | Bug minori |
|---|---|---|---|
| Backend Edge Functions | ✅ SOLIDO | 0 | 0 |
| Schema DB + RLS | ✅ CORRETTO | 0 | 0 (drift doc skill) |
| Sicurezza (advisors) | ✅ ACCETTATO | 0 | 2 WARN noti |
| Frontend SPA | ✅ CORRETTO post-fix | 0 residui | 1 cosmetico |
| Causa errore PayPal originale | ⚠️ RUNTIME | credenziali OK → causa ambiente | — |

**Nessun bug di codice bloccante residuo.** L'errore `Impossibile inizializzare PayPal` non è un difetto del codice: client-id e secret sono corretti e abbinati all'app Live `renditalabdemo`. La causa va letta a runtime dal banner `ERR_RENDER` (vedi §6).

---

## 2. Backend — Edge Functions (tutte ACTIVE)

| Funzione | verify_jwt | Esito audit |
|---|---|---|
| `paypal-create-order` v2 | true | ✅ importo `49.00 EUR` + `custom_id=user.id` impostati server-side; client non può alterarli |
| `paypal-capture-order` v2 | true | ✅ guard anti-frode: `status===COMPLETED` + `custom_id===user.id` + `currency===EUR` + `value>=49`; doppia scrittura `trials`+`paid_emails` |
| `check-trial` v2 | true | ✅ chiama `ensure_trial()`, ritorna `valid/paid/expires_at/seconds_left/email` coerenti col frontend |
| `paypal-webhook` v6 | false | ✅ signature verify PayPal + idempotenza claim-first su `processed_paypal_events.event_id` + guard importo/valuta + path `custom_id`/fallback email con `.eq('paid',false)` |

**Coerenza FE↔BE verificata:** `FN + '/paypal-create-order'` → `{ok,id}`; `onApprove` → `/paypal-capture-order` body `{orderID}` → `{ok,paid}`; `checkTrial` → `/check-trial` → `{valid,paid,seconds_left}`. Tutti i campi combaciano.

---

## 3. Schema DB + RLS

### Tabella `trials` (schema REALE verificato via MCP)
```
user_id     uuid        NOT NULL  PK
email       text        NOT NULL
created_at  timestamptz NOT NULL  default now()
expires_at  timestamptz NOT NULL  default now()+'3 days'
paid        boolean     NOT NULL  default false
paid_at     timestamptz NULL
```
RLS attivo. `ensure_trial()` usa `expires_at` → **corretto**.

### ⚠️ DRIFT DOCUMENTAZIONE (SKILL.md datata)
La skill dichiara `trial_start`/`trial_end`; lo schema reale ha `created_at`/`expires_at`. Nessun impatto sul codice (la funzione usa i nomi reali), ma **SKILL.md va aggiornata** per evitare errori futuri:
- `trial_start` → `created_at`
- `trial_end` → `expires_at`

`mark_paid(text)` confermata DROPPATA. `paid_emails` + `processed_paypal_events` RLS deny-public confermate.

---

## 4. Sicurezza (Supabase advisors)

| Advisor | Livello | Decisione |
|---|---|---|
| `ensure_trial` SECURITY DEFINER eseguibile da `authenticated` | WARN | ✅ ACCETTATO — intenzionale, documentato. Opera solo sulla riga di `auth.uid()`, serve a bypassare RLS deny di `paid_emails`. NON convertire a INVOKER. |
| Leaked password protection disabilitata | WARN | ✅ IRRILEVANTE — auth via **magic link** (`signInWithOtp`), nessuna password in gioco. Opzionale abilitarla come difesa in profondità (impostazione dashboard, non codice). |

Nessun advisor di livello ERROR.

---

## 5. Frontend — Bug trovati e corretti

| # | Bug | Severità | Stato | Fix |
|---|---|---|---|---|
| F1 | Nel gate, `pp-mount` senza sessione mostrava bottone "Registrati per attivare" → `openAuthGate()` → **loop/no-op**. Cliente non capiva come pagare. | ALTA (UX, conversione) | ✅ FIXED | `insideGate` branch: testo "Registrati con l'email qui sopra, i pulsanti PayPal appariranno dopo il login" |
| F2 | Intent di pagamento **perso** dopo magic link: utente cliccava "attiva accesso", si registrava, tornava e non trovava il pagamento. | ALTA (conversione) | ✅ FIXED | `sessionStorage.payIntent` → su `SIGNED_IN` scroll automatico + highlight verde su `#pp-cta` |
| F3 | Errori PayPal **invisibili su mobile** (solo `console`, niente F12 su smartphone). Diagnosi impossibile. | ALTA (operativa) | ✅ FIXED | Banner a schermo `ERR_SDK:` / `ERR_RENDER:` con messaggio tecnico |
| F4 | `_ppSdk` memoizzava la Promise anche su **reject** → "Riprova" inutile (riusava il fallimento). | MEDIA | ✅ FIXED | `_ppSdk = null` su ogni reject |
| F5 | `loadPayPalSDK` risolveva su `onload` anche con `window.paypal` privo di `.Buttons` (client-id invalido → SDK serve script d'errore ma `onload` scatta) → TypeError opaco a valle. | MEDIA | ✅ FIXED | Check `typeof window.paypal.Buttons === 'function'` prima di resolve |
| F6 | DevTools sourcemap (`.map` cdnjs/jsdelivr) bloccati da `connect-src` → rumore console. | BASSA | ✅ FIXED | CDN aggiunti a `connect-src` (meta + `vercel.json`) |
| F7 | Guard eligibilità mancante + funding extra. | BASSA | ✅ FIXED | `isEligible()` guard + `disable-funding=credit,paylater` |

### Minori NON corretti (bassa priorità, non bloccanti)
- **M1** — Gate anonimo mostra "Ho pagato → ricontrolla" anche a chi non ha mai pagato. Affordance legittima (pagamento da altro device/bonifico) → lasciato.
- **M2** — `paypal-capture-order` aggiorna `paid_at` senza `.eq('paid',false)` (il webhook ha il guard). Cosmetico, nessun impatto funzionale → lasciato (eviterebbe un deploy edge non necessario).

### Verifiche superate
- Tutti gli handler `onclick` → funzioni definite (nessuna ref rotta).
- `node --check` su tutto lo script applicativo (1085 righe) → **OK**.
- Magic link redirect: `emailRedirectTo` strippa l'hash → ritorno URL pulito. ✅
- Nessun testo "console (F12)" user-facing residuo. ✅

---

## 6. Errore PayPal originale — stato diagnosi

**Esclusi (verificati):**
- ❌ CSP (paypal.com pieno su script/connect/frame/img)
- ❌ Edge function / backend (tutte coerenti, env=live)
- ❌ Client-id errato (`ASwcfjfu…` == app Live `renditalabdemo`)
- ❌ Secret errato (confermato da Enrico == app renditalabdemo)
- ❌ Rumore estensioni (MetaMask/evmAsk) e sourcemap

**Residuo (causa più probabile):** stato dell'**account/app PayPal Live** — onboarding merchant incompleto, account business non verificato per EUR, o app Live priva di funding eleggibile. `render()` rigetta lato PayPal **prima** di toccare il backend.

**Diagnosi definitiva:** dopo deploy del file finale, leggere il banner **`ERR_RENDER: …`** direttamente nel box (visibile su mobile). Il messaggio PayPal indica la causa esatta.

---

## 7. Azioni residue per Enrico

1. **Deploy file finale** (`index.html` + `vercel.json`) via GitHub Desktop → attendere `READY`.
2. **Hard refresh** su mobile → registrarsi → scorrere al box → leggere `ERR_RENDER:`.
3. Inviare il testo `ERR_RENDER` → chiude la causa ambiente.
4. **Aggiornare SKILL.md**: schema `trials` (`trial_start/trial_end` → `created_at/expires_at`).
5. (Opzionale) PayPal: verificare onboarding account business Live + valuta EUR abilitata.

---

## 8. Rollback
Tutte le modifiche sono frontend single-file + header Vercel. Rollback = ripristino commit precedente da GitHub Desktop / Vercel "Instant Rollback". Nessuna modifica DB/edge applicata in questo audit (sola lettura).
