# Palvelureppu-analytiikka — deployment

Palvelureppu Oy:n oma Flarelytics-instanssi. Eristetty kalle.works-tilistä:
oma Cloudflare-tili, oma Analytics Engine -data, oma worker + domain.

- **Cloudflare-tili:** `abcb70ce41324ef39d54ee3f76fbf04e` (Palvelureppu Oy)
- **Worker:** `palvelureppu-analytiikka`
- **Domain:** `analytiikka.palvelureppu.fi` (tracker `/tracker.js`, `/track`, `/query`, `/health`)
- **Upstream:** `git remote upstream` = `kalle-works/flarelytics` (vedä parannukset: `git fetch upstream && git merge upstream/main`)

`wrangler.toml` on gitignored — tämä dokumentti on sen kanoninen lähde. Salaisuudet
asetetaan `wrangler secret put` -komennolla, eivät tiedostoon.

## Tila (deployattu 2026-06-07)

Worker on **live**: `analytiikka.palvelureppu.fi` (custom domain, sertti aktiivinen).
Verifioitu: `/health`, `/config`, `/tracker.js` (200), `/track` (204 validi origin,
403 väärä origin).

- KV `SITE_CONFIG` namespace id: **`2bed91dd1e37478496b136273ec91774`**
  (title `palvelureppu-analytiikka-site-config` — oma, ei jaettu muiden palveluiden kanssa)
- Asetetut secretit: `QUERY_API_KEY` ✓, `CF_ACCOUNT_ID` ✓
- **Puuttuu: `CF_API_TOKEN`** → `/query`-lukupolku ja dashboard eivät vielä toimi
  (trackaus toimii silti). Luo dashboardissa
  https://dash.cloudflare.com/profile/api-tokens → Custom token →
  *Account > Account Analytics > Read* (Palvelureppu-tili) → aseta:
  `echo '<token>' | npx wrangler secret put CF_API_TOKEN`. Tämän jälkeen `/health` → `healthy`.

## Kertaluontoinen pystytys

Web-analytiikan minimaalinen jalanjälki: Analytics Engine luo datasetit
automaattisesti ensimmäisellä kirjoituksella → **ei D1-migraatiota, ei queueja,
ei R2:ta**. Tarvitaan vain yksi KV-namespace (valinnainen) + secretit + deploy.

```bash
export CLOUDFLARE_ACCOUNT_ID=abcb70ce41324ef39d54ee3f76fbf04e
cd packages/worker

# 1. KV (valinnainen — staattinen ALLOWED_ORIGINS toimii ilmankin, mutta tämä
#    antaa lisätä siteja + näyttää dashboardin site-listan). Kopioi tulostettu
#    id wrangler.tomlin [[kv_namespaces]]-blokkiin.
npx wrangler kv namespace create SITE_CONFIG

# 2. Salaisuudet
openssl rand -hex 16 | npx wrangler secret put QUERY_API_KEY   # talleta arvo turvallisesti
npx wrangler secret put CF_API_TOKEN     # palvelureppu-tilin token, Account Analytics: Read
npx wrangler secret put CF_ACCOUNT_ID    # abcb70ce41324ef39d54ee3f76fbf04e

# 3. Deploy + verifiointi
npx wrangler deploy
curl https://analytiikka.palvelureppu.fi/health        # → {"status":"healthy", ...}
```

> D1/Queues/R2 lisätään vasta jos vaiheen 2 enrichment-/v1-pipeline otetaan
> käyttöön. Tällöin palauta vastaavat blokit `wrangler.toml`:iin ja luo resurssit
> (`wrangler d1 create …`, `wrangler queues create …`, `wrangler r2 bucket create …`).

## Instrumentoidut palvelut (ALLOWED_ORIGINS)

palvelureppu.fi · aivot.palvelureppu.fi · asiakirjapalvelu.palvelureppu.fi ·
dokumentinhallinta.fi · dokumenttipalvelu.fi · domainreppu.fi · esityseditori.fi ·
esityspalvelu.fi · helpparibotti.fi · kalenterimuistio.fi · brand.palvelureppu.fi ·
palvelureppu.email · admin.palvelureppu.fi · tukihaku.palvelureppu.fi

Uuden sitin lisäys: lisää origin `wrangler.toml`:n `ALLOWED_ORIGINS`-listaan ja
`npx wrangler deploy`. Worker tägää sitin automaattisesti Origin-headerista (blob10).

## Dataa katsotaan

```bash
curl -H "X-API-Key: $QUERY_API_KEY" \
  "https://analytiikka.palvelureppu.fi/query?q=daily-views&period=7d&site=palvelureppu.fi"
```

## Dashboard

Deployattu palvelureppu-tilin Cloudflare Pagesiin:
**https://palvelureppu-analytiikka-dashboard.pages.dev/**

Kirjautuminen (geneerinen — syötä kentät):
- Worker URL: `https://analytiikka.palvelureppu.fi`
- API key: `QUERY_API_KEY` (asetettu secret)

Uudelleendeploy: `cd packages/dashboard && npx astro build && \
  CLOUDFLARE_ACCOUNT_ID=abcb70ce41324ef39d54ee3f76fbf04e \
  npx wrangler pages deploy dist --project-name palvelureppu-analytiikka-dashboard --branch main`

Data näkyy vasta kun `CF_API_TOKEN` on korjattu (ks. yllä) — muuten `/query` → 502.
