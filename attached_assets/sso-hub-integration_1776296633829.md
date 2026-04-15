# SSO Hub Integration

## Panoramica

Questo documento descrive il protocollo di Single Sign-On (SSO) adottato per consentire a un **hub centralizzato** — il portale che raccoglie le icone di tutte le app disponibili — e alle singole **app finali** di condividere la sessione dell'utente senza che questi debba autenticarsi più volte.

Il meccanismo si basa interamente sul JWT emesso dal backend di autenticazione. L'hub ottiene il token una volta sola al login; ogni app finale espone un endpoint pubblico che riceve il token, lo valida autonomamente contro il backend e imposta la propria sessione.

---

## Architettura

```
┌──────────────────────────────────────────────────────────┐
│                          HUB                             │
│  - Unico punto di login (email + password)               │
│  - Mostra le icone di tutte le app disponibili           │
│  - Conserva il JWT per la durata della sessione hub      │
└──────────────────┬───────────────────────────────────────┘
                   │
                   │  1. Utente clicca sull'icona di un'app
                   │  2. Hub invia POST con il JWT all'endpoint
                   │     di token-login dell'app
                   ▼
┌──────────────────────────────────────────────────────────┐
│                       APP FINALE                         │
│  - Espone POST /api/auth/token-login (endpoint pubblico) │
│  - Valida il JWT: GET /auth/user sul backend             │
│  - Imposta il cookie di sessione httpOnly                │
│  - Redirige l'utente alla pagina iniziale dell'app       │
└──────────────────────────────────────────────────────────┘
```

**Flusso passo-passo:**

1. L'utente apre l'hub e si autentica con email e password.
2. L'hub chiama `POST {AUTH_BASE_URL}/auth/token` e ottiene il JWT.
3. L'hub mostra la schermata con le icone delle app.
4. L'utente clicca su un'app.
5. L'hub invia una richiesta HTTP POST all'endpoint `/api/auth/token-login` dell'app, con il JWT nel body.
6. L'app riceve il token, lo valida chiamando `GET {AUTH_BASE_URL}/auth/user`, imposta il cookie di sessione e redirige l'utente.
7. L'utente è autenticato nell'app senza aver reinserito le credenziali.

---

## API del backend di autenticazione

Queste API sono esposte dal backend Biolaser centralizzato. Sono utilizzate sia dall'hub al momento del login sia da ciascuna app finale per validare i token ricevuti.

### POST /auth/token

Autentica l'utente con email e password e restituisce il JWT.

**Autenticazione:** nessuna.

**Richiesta:**

```http
POST {AUTH_BASE_URL}/auth/token HTTP/1.1
Content-Type: application/json

{
  "email": "utente@azienda.it",
  "password": "password",
  "version": "4.0"
}
```

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `email` | string | Sì | Viene trimmata lato server |
| `password` | string | Sì | |
| `version` | string | No | Versione client; il backend può rifiutare versioni obsolete |

**Risposta 200:**

```json
{
  "token": "<jwt>"
}
```

**Errori:**

| Codice | Causa |
|---|---|
| `401` | Credenziali non valide (`error: invalid_credentials`) |
| `500` | Impossibile creare il token (`error: could_not_create_token`) |
| `500` | Versione client non supportata (`wrongVersion: true`) |

---

### GET /auth/user

Restituisce il profilo completo dell'utente associato al JWT. Utilizzato sia al primo login (dopo aver ottenuto il token) sia da ogni app finale per validare un token ricevuto via SSO.

**Autenticazione:** `Authorization: Bearer <jwt>`

**Richiesta:**

```http
GET {AUTH_BASE_URL}/auth/user HTTP/1.1
Authorization: Bearer <jwt>
Accept: application/json
```

**Parametri query opzionali:**

| Parametro | Note |
|---|---|
| `c_role` | Forza il ruolo corrente se l'utente possiede un ruolo attivo con quel nome |

**Risposta 200:**

```json
{
  "user": {
    "id": 123,
    "firstname": "Mario",
    "lastname": "Rossi",
    "email": "mario.rossi@azienda.it",
    "type": "admin",
    "store_id": 10,
    "tenant": "Italia",
    "alias": "Mar.Ros",
    "roles": [],
    "perm": [],
    "permissions": [],
    "config": []
  }
}
```

| Campo | Tipo | Descrizione |
|---|---|---|
| `id` | number | Identificativo univoco dell'utente |
| `firstname` / `lastname` | string | Nome e cognome |
| `email` | string | Indirizzo email |
| `type` | string | Tipo utente (es. `admin`, `user`) |
| `store_id` | number \| null | Punto vendita associato |
| `tenant` | string | Nome del tenant/area |
| `alias` | string | Abbreviazione visualizzata nell'interfaccia |
| `roles` | array | Ruoli attivi dell'utente |
| `permissions` | array | Lista distinta dei nomi di permesso |
| `config` | array | Configurazione client per ruolo/tenant/store |

**Errori:**

| Codice | Causa |
|---|---|
| `401` | Token assente, scaduto o invalido |
| `403` | Token valido ma utente non autorizzato |

---

### POST /invalidate

Invalida il JWT lato server. Va chiamato al logout.

**Autenticazione:** `Authorization: Bearer <jwt>`

**Richiesta:**

```http
POST {AUTH_BASE_URL}/invalidate HTTP/1.1
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "token": "<jwt>",
  "reason": "logout"
}
```

**Risposta 200:**

```json
{
  "message": "OK"
}
```

> Anche se la chiamata fallisce, il client deve comunque eliminare il token dalla propria sessione locale.

---

## Endpoint SSO delle app finali: `POST /api/auth/token-login`

Ciascuna app finale deve esporre questo endpoint. Riceve il JWT dall'hub, lo valida tramite `GET /auth/user` e imposta il proprio cookie di sessione. L'endpoint è **pubblico** (non richiede autenticazione preesistente).

### Modalità di risposta

L'endpoint supporta due modalità, determinate dall'header `Accept` della richiesta:

| `Accept` | Modalità | Risposta in caso di successo |
|---|---|---|
| `application/json` | JSON | `200` con body JSON + cookie impostato |
| qualsiasi altro valore | Form/redirect | `302` redirect alla pagina iniziale + cookie impostato |

---

### Modalità 1 — JSON

Usata quando l'hub chiama l'endpoint tramite una richiesta HTTP programmatica (es. `fetch`, `XMLHttpRequest`, `curl`, client HTTP da qualsiasi linguaggio). Dopo aver ricevuto la risposta con successo, l'hub naviga all'URL restituito.

**Richiesta:**

```http
POST https://{app-hostname}/api/auth/token-login HTTP/1.1
Content-Type: application/json
Accept: application/json

{
  "token": "<jwt>",
  "redirect_to": "/dashboard"
}
```

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `token` | string | Sì | JWT ottenuto da `POST /auth/token` |
| `redirect_to` | string | No | Path interno verso cui navigare dopo il login. Default: `/overview`. Deve iniziare con `/` (path relativi all'app). |

**Risposta 200 — successo:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: auth_token=<jwt>; HttpOnly; Path=/; SameSite=Lax; Max-Age=43200

{
  "success": true,
  "redirectUrl": "/dashboard"
}
```

Il cookie `auth_token` viene impostato dall'app. L'hub deve navigare a `{APP_URL}{redirectUrl}` dopo aver ricevuto la risposta.

---

### Modalità 2 — Form POST

Usata per navigazioni browser cross-origin dove non è configurato CORS. Il browser esegue un normale invio di form: la risposta `302` imposta il cookie e naviga automaticamente all'interno dell'app. Non richiede codice aggiuntivo lato hub oltre alla costruzione del form.

**Richiesta:**

```http
POST https://{app-hostname}/api/auth/token-login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

token=<jwt>&redirect_to=%2Fdashboard
```

**Risposta 302 — redirect:**

```http
HTTP/1.1 302 Found
Location: /dashboard
Set-Cookie: auth_token=<jwt>; HttpOnly; Path=/; SameSite=Lax; Max-Age=43200
```

Il browser segue automaticamente il redirect e l'utente si trova già autenticato nell'app.

---

### Errori

Le risposte di errore seguono la stessa logica di modalità:

- In modalità **JSON** (`Accept: application/json`): body JSON con `success: false` e `error`.
- In modalità **Form/redirect**: redirect a `{APP_URL}/login?error=<codice>`.

| Codice HTTP | Causa | `error` in query string (form) |
|---|---|---|
| `400` | `token` mancante o vuoto | `token_missing` |
| `401` | Token non valido o scaduto (risposta 4xx da `/auth/user`) | `token_invalid` |
| `503` | Backend di autenticazione non raggiungibile o in errore 5xx | `auth_unavailable` |
| `500` | Errore interno dell'app | `server_error` |

**Esempio risposta errore 401 (modalità JSON):**

```json
{
  "success": false,
  "error": "Token non valido o scaduto"
}
```

---

## Flusso di chiamate HTTP completo

```
HUB                                    APP FINALE           BACKEND AUTH
 │                                          │                     │
 │── POST /auth/token ─────────────────────────────────────────►│
 │◄─ 200 { token: "<jwt>" } ───────────────────────────────────│
 │                                          │                     │
 │  [utente clicca sull'icona dell'app]     │                     │
 │                                          │                     │
 │── POST /api/auth/token-login ──────────►│                     │
 │   { token: "<jwt>" }                    │                     │
 │                                          │── GET /auth/user ──►│
 │                                          │   Authorization: Bearer <jwt>
 │                                          │◄─ 200 { user: {...} }
 │                                          │                     │
 │◄─ 200 { success: true, redirectUrl } ──│                     │
 │   Set-Cookie: auth_token=<jwt>          │                     │
 │                                          │                     │
 │  [hub naviga a {APP_URL}/dashboard]      │                     │
 │──────────────────────────────────────── GET /dashboard ──────►│ (cookie inviato)
```

---

## Considerazioni di sicurezza

### HTTPS obbligatorio

Il JWT trasmesso nel body è una credenziale di accesso. Tutta la comunicazione tra hub e app deve avvenire **esclusivamente su HTTPS** in produzione. Il cookie di sessione deve essere impostato con il flag `Secure` attivo in produzione.

### Il JWT non va mai nell'URL

Non usare `GET /api/auth/token-login?token=<jwt>`. Il token nella query string:
- viene registrato nei log del server web;
- viene salvato nella cronologia del browser;
- può essere catturato dall'header `Referer` nelle navigazioni successive.

Usare sempre POST con il token nel body.

### Protezione da open redirect

Il parametro `redirect_to` deve essere validato dall'app finale. Sono accettati solo path interni (es. `/dashboard`, `/reports`). Valori assoluti come `https://sito-esterno.com` o protocol-relative come `//sito-esterno.com` devono essere ignorati e sostituiti con il path di default dell'app.

### Validazione obbligatoria del token

L'app finale non deve mai fidarsi del JWT senza verificarlo. Prima di impostare il cookie di sessione, l'app deve sempre chiamare `GET /auth/user` e accettare il token solo in caso di risposta `200` con profilo utente valido.

### CORS (solo per modalità JSON cross-origin)

Se hub e app si trovano su domini diversi e si usa la modalità JSON (con `fetch`), l'app deve configurare gli header CORS per il solo endpoint `/api/auth/token-login`, specificando l'origine dell'hub come valore di `Access-Control-Allow-Origin`. La modalità Form POST non richiede alcuna configurazione CORS.

### Durata del cookie di sessione

Il cookie di sessione ha una durata massima di 12 ore (720 minuti). L'app deve intercettare le risposte di errore del backend con codici come `token_expired`, `token_absent` o `token_invalid` e procedere al logout automatico dell'utente.

---

## Riepilogo modalità di integrazione

| Scenario | Modalità consigliata |
|---|---|
| Hub e app sullo stesso dominio | JSON (`Accept: application/json`) |
| Hub e app su domini diversi, CORS configurato | JSON con header `Access-Control-Allow-Origin` configurato |
| Hub e app su domini diversi, senza CORS | Form POST (`Content-Type: application/x-www-form-urlencoded`) |
| Test da terminale o sistema server-side | JSON (`Accept: application/json`) |
