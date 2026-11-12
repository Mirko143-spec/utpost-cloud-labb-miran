# Beslutsdokument: hostingval

*Ett av teamets sex beslutsdokument. Skrivs i M6, hålls levande. Samma mall som `api-stil.md`, `databas.md` och `testing.md`. Det här dokumentet examinerar Cloud 1 (redogöra för hostinglösningar som AWS, Azure, Google Cloud) – skriv så att en läsare utan Azure-konto förstår valet.*

**Datum:**
**Beslut:** *(en eller två meningar: var kör Utpost i staging, var ska den köra i produktion, och vad är skillnaden?)*

## Bakgrund
*Vad har vi när M5 är klar: två images i GHCR taggade med sha, en pipeline som bygger dem, compose lokalt. Vad saknas för att någon utanför teamet ska kunna öppna appen? Vilka krav ställer kursen (M6: automatisk deploy till staging vid merge, secrets ur koden, publik URL)?*

## Vad vi faktiskt hyr
*Fyll i tabellen med era egna värden ur Azure-portalen eller `az containerapp show`. Kolumnen "nivå" = IaaS / PaaS / SaaS / serverless (klippet *Molnet: vad det är och vad man hyr*).*

| Del | Tjänst | Nivå | Vem sköter uppdateringar/skalning | Vad det kostar oss i staging |
|---|---|---|---|---|
| Klient + API | Azure Container Apps, plan Consumption, en app (`utpost-staging`) där alla fyra containrar körs bredvid varandra (sidecars) | PaaS (vi lämnar in containrar, Azure kör dem) | Microsoft sköter servrar, OS och skalning (0–1 replica, `minReplicas: 0`); vi sköter imagerna och vilken sha som körs | 0 kr hittills (Cost analysis, month-to-date 12/11). Kostnadsdata släpar ett dygn, och vi ligger under gratiskvoten. I dag: 21 anrop, högst 1 replica. Appen begär 1,75 vCPU + 3,5 GiB. Gratiskvoten (180 000 vCPU-s + 360 000 GiB-s per månad och prenumeration) räcker till ~28,6 h vaken per månad om vi vore ensamma, men delat av 29 är det ~1 h var |
| Postgres | `postgres:16` som sidecar i samma app, ingen volym | ingen tjänst – vår egen container | vi – och det finns ingen backup: databasen försvinner när repliken stängs | ingår i raden ovan (0,5 vCPU / 1 GiB) |
| MongoDB | `mongo:8.2` som sidecar i samma app, ingen volym | ingen tjänst – vår egen container | vi – ingen backup, samma som Postgres | ingår i raden ovan (0,5 vCPU / 1 GiB) |
| Images | GHCR (`ghcr.io/mirko143-spec/utpost-cloud-labb-miran-api`/`-client`, publika) | SaaS | GitHub lagrar dem, vi bygger och taggar dem (`sha-<7>`) | 0 kr – publika paket är gratis |
| Deploy | GitHub Actions → `az containerapp update` med OIDC (ingen lösenordshemlighet) | SaaS | GitHub kör jobbet på sina runners, vi skriver det | 0 kr – publikt repo, Actions-minuter gratis (~99 s per deploy) |

## Alternativ vi jämförde
*Minst tre riktiga alternativ, och minst två leverantörer. Förslag: Azure Container Apps (det vi gjorde) · Azure App Service for Containers · Google Cloud Run · AWS ECS på Fargate (App Runner tar inte längre nya kunder) · en VM (IaaS) med docker compose · Render/Railway (PaaS utanför de tre stora). För varje: vad hade ändrats i repot, vad hade kostat, vad hade vi fått/förlorat?*

| Alternativ | Vad ändras i repot | Kostnad (staging) | För | Emot |
|---|---|---|---|---|
| | | | | |

## Staging och produktion är inte samma sak
*I staging kör databaserna som sidecars utan volym och seedas vid varje kallstart. Vad krävs för produktion? Managed databas (vilken, vad kostar den), backup, en app som inte skalar till noll, egen domän och certifikat, loggar. Skriv vad ni skulle byta och vad som INTE behöver bytas (imagen? pipelinen?).*

## Inlåsning
*Om Azure försvann i morgon: vilka filer i repot ändras, och vilka ändras inte? Räkna upp dem. Det är svaret på hur portabel lösningen är – och argumentet för att bygga på image + pipeline i stället för plattformens egna byggverktyg.*

**Finns bara för Azure. De här byts om vi flyttar, till exempel till Cloud Run eller ECS:**
- `infra/azure/staging.yaml` – appens definition (containrar, resurser, secrets, skalning 0–1)
- `infra/azure/skapa-staging.sh` – skapar miljön och appen en gång
- `infra/azure/identitet.sh` – app registration + federerad identitet (OIDC) + Contributor på resursgruppen
- jobbet `deploy` i `.github/workflows/ci.yml` – `azure/login` + `az containerapp update`
- variablerna på GitHub-miljön `azure-staging` (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RG`, `AZURE_APP`, `AZURE_URL`)

**Är likadant i alla moln:**
- `api/Dockerfile`, `client/Dockerfile` och imagerna i GHCR – ingenting i imagen vet att den kör i Azure
- `client/nginx.conf.template` – adressen till api läses ur `API_UPSTREAM` när containern startar
- `api/src/` – all applikationskod
- `.env.example` och `compose.yaml` – lokal utveckling
- jobben Kvalitet, Bygg, Integration och Image i pipelinen

**Värdena som skiljer compose (lokalt) från Azure (staging):**

| Värde | compose | Azure staging |
|---|---|---|
| `API_UPSTREAM` | `api:4000` (egen container, Dockers DNS) | `localhost:4000` (sidecar i samma app) |
| `DATABASE_URL` | `postgres://utpost:utpost@postgres:5432/utpost` | `…@localhost:5432/…`, lösenordet slumpat av `skapa-staging.sh` och lagrat som secret |
| `MONGO_URL` | `mongodb://utpost:utpost@mongo:27017/…` | `…@localhost:27017/…`, slumpat lösenord som secret |
| `JWT_SECRET` | `utpost-dev-secret-byt-i-molnet` i klartext | secret `jwt-secret` i Azure (slumpad, roterad 12/11 med `az containerapp secret set` + omstart) |
| `SEED_ON_START` | `'1'` (harmlöst lokalt, seedar bara om tom) | `1` – varje kallstart ger tom databas |

Det som låser oss är alltså fyra filer och några värden. Allt annat följer med oförändrat.

## Konsekvenser
*Vad kostar valet i tid och pengar? Kallstarten när appen skalat till noll. Gratiskvoten som delas av alla i prenumerationen. Hemligheter som nu lever på två ställen (lokalt `.env`, i molnet secrets) – hur roteras de? Vem i teamet har tillgång till portalen?*

**Kallstarten:** första anropet efter skapandet (12/11) tog 13,4 s till `{"ok":true,…}` (`time until curl …/api/health`). Appen var nyss skapad, så den var troligen delvis igång redan. En deploy som väcker appen från noll tar enligt mallen ungefär en minut extra (fyra containrar startar, api väntar på Postgres och seedar sedan). I loggen syns `boot: väntar på Postgres (1/30)` → `boot: tom databas – seedar` → `Seed klar: 12 användare, 40 guider, 200 turer` → `API lyssnar`. Varje kallstart ger alltså en nyseedad databas. Det som skapats sedan förra starten är borta, till exempel användaren från rotationstestet. Det duger i staging, där vi bara vill se att rätt kod startar. I produktion duger det inte: där måste data överleva omstarter, och därför behövs en managed databas med backup och en app som inte skalar till noll.

**Gratiskvoten:** 1,75 vCPU + 3,5 GiB per vaken replica ger ~28,6 h per månad för hela prenumerationen, delat av 29 personer. `minReplicas` ska stå kvar på 0.

**Hemligheter:** finns bara i Azure (`az containerapp secret list` visar namnen, inte värdena). Rotation = `secret set` + `revision restart`. En secret är ingen revision, så bytet märks först när containrarna startat om (55 s i vårt test, fram till dess svarade den gamla repliken med 200).

**Skrivet av:**
