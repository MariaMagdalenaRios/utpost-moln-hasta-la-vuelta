# Beslutsdokument: hostingval

*Ett av teamets sex beslutsdokument. Skrivs i M6, hålls levande. Samma mall som `api-stil.md`, `databas.md` och `testing.md`. Det här dokumentet examinerar Cloud 1 (redogöra för hostinglösningar som AWS, Azure, Google Cloud) – skriv så att en läsare utan Azure-konto förstår valet.*

**Datum:** 2026-10-30
**Beslut:** *(en eller två meningar: var kör Utpost i staging, var ska den köra i produktion, och vad är skillnaden?)*

## Bakgrund
*Vad har vi när M5 är klar: två images i GHCR taggade med sha, en pipeline som bygger dem, compose lokalt. Vad saknas för att någon utanför teamet ska kunna öppna appen? Vilka krav ställer kursen (M6: automatisk deploy till staging vid merge, secrets ur koden, publik URL)?*

## Vad vi faktiskt hyr
Staging ligger i `swedencentral` som en Azure Container App med fyra containrar. Appen begär
`0,25 + 0,5 + 0,5 + 0,5 = 1,75 vCPU` och `0,5 + 1 + 1 + 1 = 3,5 GiB`, och kan skala mellan
0 och 1 replica. Den 30 oktober 2026 visade Azure Monitor 25 requests under dagen. En kort
övergång till två replicas syntes i samband med revisionsbytet.

| Del | Tjänst | Nivå | Vem sköter uppdateringar/skalning | Vad det kostar oss i staging |
|---|---|---|---|---|
| Klient + API | Azure Container Apps på Consumption-planen, en app och en revision med fyra containrar | Serverless/PaaS | Microsoft sköter servrar och plattformsskalning; appen går 0–1 replica och vi ansvarar för imagerna | `0,00 USD` synligt/beräknat så långt. Cost Management-frågan hade inga bokförda rader den 30 oktober eftersom kostnadsdata är fördröjd; dagens användning ligger under gratiskvoten. Efter gratiskvoten kostar en helt vaken timme `1,75 × 3 600 × 0,000024 + 3,5 × 3 600 × 0,000003 = 0,189 USD` |
| Postgres | `postgres:16` som sidecar i samma Container App, utan volym | Ingen separat tjänst – vår container | Vi sköter uppdatering och backuper; det finns ingen backup i staging | Ingår i resurserna och kostnaden för raden Klient + API |
| MongoDB | `mongo:8.2` som sidecar i samma Container App, utan volym | Ingen separat tjänst – vår container | Vi sköter uppdatering och backuper; det finns ingen backup i staging | Ingår i resurserna och kostnaden för raden Klient + API |
| Images | GitHub Container Registry (GHCR) | SaaS | GitHub lagrar paketen; vi publicerar sha-taggade images från Actions | `0 USD`; repot och containerpaketen är publika, och lagring/bandbredd för offentliga paket är kostnadsfri enligt GitHubs nuvarande prisregler |
| Deploy | GitHub Actions → `az containerapp update` med OIDC | SaaS | GitHub kör jobbet, vi äger workflow, miljövariabler och rolltilldelning | `0 USD`; standardrunnern på Ubuntu är kostnadsfri för ett publikt repo |

Källor: [Azure Container Apps – Consumption pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/),
[GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
och [GitHub Packages billing](https://docs.github.com/en/billing/concepts/product-billing/github-packages).

## Alternativ vi jämförde
*Minst tre riktiga alternativ, och minst två leverantörer. Förslag: Azure Container Apps (det vi gjorde) · Azure App Service for Containers · Google Cloud Run · AWS ECS på Fargate (App Runner tar inte längre nya kunder) · en VM (IaaS) med docker compose · Render/Railway (PaaS utanför de tre stora). För varje: vad hade ändrats i repot, vad hade kostat, vad hade vi fått/förlorat?*

| Alternativ | Vad ändras i repot | Kostnad (staging) | För | Emot |
|---|---|---|---|---|
| | | | | |

## Staging och produktion är inte samma sak
*I staging kör databaserna som sidecars utan volym och seedas vid varje kallstart. Vad krävs för produktion? Managed databas (vilken, vad kostar den), backup, en app som inte skalar till noll, egen domän och certifikat, loggar. Skriv vad ni skulle byta och vad som INTE behöver bytas (imagen? pipelinen?).*

## Inlåsning
Azure-specifikt och behöver ersättas vid en flytt:

- `infra/azure/staging.yaml` beskriver Container Apps-resursen, sidecars, den externa ingressen och
  skalningen till noll.
- `infra/azure/skapa-staging.sh` anropar Azure CLI för resursgrupp, Container Apps-miljö och app.
- `infra/azure/identitet.sh` skapar Entra-app, service principal, federerad GitHub-identitet och
  Azure RBAC-rolltilldelning.
- Jobbet `deploy` i `.github/workflows/ci.yml` använder `azure/login`, `az containerapp update` och
  Azures revisionssuffix. GitHub-miljön innehåller också Azures klient-, tenant-, prenumerations-,
  resursgrupps- och app-ID:n.

Portabelt och kan behållas på Cloud Run, ECS eller en annan containertjänst:

- `api/Dockerfile` och `client/Dockerfile`
- `client/nginx.conf.template`
- `.env.example`, `compose.yaml`, `api/src/` och `shared/`
- de byggda OCI-imagerna och de vanliga CI-jobben Kvalitet, Bygg, Integration och Image

Det som skiljer mellan miljöerna är i huvudsak värden, inte applikationskod:

- `API_UPSTREAM` är `api:4000` i compose och `localhost:4000` i Azure, eftersom containrarna delar
  localhost i samma Container App.
- `DATABASE_URL` använder `localhost:5433` vid lokal npm-utveckling, `postgres:5432` i compose och
  `localhost:5432` i Azure.
- `MONGO_URL` använder `mongo` som värdnamn i compose och `localhost` i Azure.
- `JWT_SECRET` är ett utvecklingsvärde i `.env`, men refereras som Azure-secret `jwt-secret` i
  staging. Rotation sker med `az containerapp secret set` följt av en omstart.
- `SEED_ON_START` är `0` i exempelkonfigurationen och `1` i compose samt Azure, där den tomma
  stagingdatabasen seedas vid start.

Slutsatsen är att imagen inte behöver veta att den kör i Azure. En flytt kräver ny
plattformskonfiguration och motsvarande deploysteg, men inte nya Docker-images eller ändrad
applikationskod.

## Konsekvenser
*Vad kostar valet i tid och pengar? Kallstarten när appen skalat till noll. Gratiskvoten som delas av alla i prenumerationen. Hemligheter som nu lever på två ställen (lokalt `.env`, i molnet secrets) – hur roteras de? Vem i teamet har tillgång till portalen?*

**Kallstart och gratiskvot:** Eftersom appen kör `minReplicas: 0` startar alla fyra containrarna
först vid första requesten. Vid kontrollen den 30 oktober svarade nginx med `502` några sekunder
medan API:t startade och databaserna seedades; därefter svarade både `/api/health` och GraphQL.
Deployjobbet väntar upp till fem minuter, vilket täcker den observerade kallstarten med god marginal.

Azure Container Apps ger per prenumeration och månad 180 000 vCPU-sekunder, 360 000 GiB-sekunder
och 2 miljoner requests. Med appens begärda storlek räcker CPU- och minneskvoten till:

```text
min(180 000 / 1,75, 360 000 / 3,5) / 3 600
= 102 857,14 sekunder / 3 600
≈ 28,57 vakna timmar per månad
```

Om klassen har `N` identiska stagingappar som alla är vakna samtidigt blir andelen ungefär
`28,57 / N` vakna timmar per app och månad. Antalet lag syns inte genom vår åtkomst till
resursgruppen, så värdet för `N` måste fyllas i av kursen eller teamet. Apparna skalas normalt till
noll och delar dessutom requestkvoten; vår observerade användning på 25 requests är långt under
2 miljoner.

Vid rotation byttes `jwt-secret` utan ny build. En gammal token gav `200` både före bytet och direkt
efter `secret set`; först när revisionen startades om gav samma token `401`. SHA:n var oförändrad
under hela rotationen.

**Skrivet av:**
