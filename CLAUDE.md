# Tibor Evergreen Webinar — Project Handoff / Onboarding

> Ovaj dokument uvodi novog člana tima (i Claude-a) u projekat. Sadrži: šta je projekat,
> šta je SVE urađeno, kako funkcioniše, i šta JOŠ treba uraditi/istestirati. Pisano da
> nema mesta za greške. (Jezik: srpski + tehnički termini.)

---

## 1. Šta je projekat

Evergreen webinar funnel za klijenta **Tibor** (brend "Edit u Novac", video editing edukacija).
Snimljeni webinar koji se ponaša kao **live**, kreće **svaki dan u 20:00 Europe/Belgrade**.
Kampanja (reklame) ide svaki dan; svako ko se prijavi gleda "današnji" webinar.

Cilj naše izmene: **per-osoba countdown** na bonus stranicama, vezan za **dan kad se osoba
optin-ovala** (njihov webinar dan), preko `?datum=` URL parametra.

Hosting stranica: **GoHighLevel (GHL)**, domen `uzivotrening.editunovac.com`.
Plaćanje: **Stripe** (subscription, 97.99 EUR / 3 meseca). Isporuka pristupa: **Whop**.
Automatizacija posle plaćanja: **Zapier** (NIJE u ovom repo-u). Tracking: **Meta pixel** + **Hyros** (Hyros još nije postavljen).

---

## 2. Fajlovi u ovom repo-u

| Fajl | GHL stranica | Šta je |
|---|---|---|
| `evergreen-webinar-start.html` | `/start` | Bonus strana sa **WEBINARA** (live). Ima **brze bonuse** (prozor 20:00–21:34) + 4-dnevnu ponudu. |
| `evergreen-webinar-snimka.html` | `/snimka` | **Replay** strana (šalje se posle webinara). Ima **email gate** (30s gledanja → traži email) + 4-dnevnu ponudu. Bez brzih bonusa. |
| `evergreen-webinar-repitch.html` | `/ponuda` | **Re-pitch** strana. 4-dnevna ponuda, bez brzih bonusa. |
| `backup/` | — | Originalne (pre-izmena) verzije za rollback. NE pushovati na GHL. |

Stranice su GHL **Custom HTML/JS blokovi** — paste-uje se CEO sadržaj fajla u taj blok na stranici.

---

## 3. Kako radi countdown (`?datum=` contract) — SUŠTINA

Svaka stranica ima `<script>` tajmer sa istim helper-ima (`belgradeEpoch`, `parseDatum`,
`lastWebinarDay`, `fmtBg`).

**Anchor model** (za webinar dan `Y-M-D`):
- `fastBonusEnd = belgradeEpoch(Y,M,D, 21,34)` — **samo start.html bez parametra** (live brzi bonusi)
- `offerEnd = belgradeEpoch(Y,M,D+4, 23,59,59)` — kraj 4-dnevne ponude

**Odakle webinar dan:**
- **Sa `?datum=DD_MM_YYYY`** (mejl/WA linkovi) → taj datum. Parser guta i `DD_MM_YY` (2-cifre).
- **Bez parametra** (bare /start, link sa snimke) → **poslednjih prošlih 20:00 Beograd** (`lastWebinarDay`).

**`belgradeEpoch`** računa pravi Beograd offset preko `Intl` (CEST leto +2 / CET zima +1).
**NIKAD ne hardkodovati `+02:00`** — lomi se zimi.

**start.html grananje:**
- **Sa param** → 4-dnevna ponuda (bez brzih bonusa).
- **Bez param**:
  - 20:00–21:34 → brzi bonusi (fast)
  - posle 21:34 → `location.replace` na `?datum=<taj_dan>` → 4-dnevna ponuda (tick proverava svake sekunde, pa se i otvoren tab sam prebaci)

**Sticky datum (localStorage key `euo_datum`):** svaki učitaj sa validnim `?datum=` ga snima
(sve 3 strane). Bare /start: ako postoji sačuvani datum (i nije današnji live) → redirect na
NJEGOV sačuvani datum (ne na trenutni webinar). Tako povratnik uvek dobije svoj countdown.
Re-optin slučaj: istekao sačuvani datum NE blokira aktuelan live (dobije nove brze bonuse).

**Expired stanje:** kad `now > offerEnd` → countdown 00:00:00:00, CTA posivi ("Ponuda je istekla").

**`19:45 → sutra` pravilo:** živi **SAMO u Make-u** (optin-time pravilo: ko se optin-uje posle
19:45 dobija sutrašnji datum). NE ponavlja se na stranici (viewing-time).

---

## 4. Backend: fulfillment posle plaćanja (Zapier — van repo-a)

Tok: **Stripe New Subscription → Filter (cena) → Code by Zapier (JS) → Gmail (pristup mejl)**

- **Filter:** cena u `[47, 39.99, 97, 141, 97.99]`. Realna cena = **97.99** (oba Stripe linka).
- **Code (JS):** poziva **Whop API** → pravi **besplatan, jednokratan** Whop plan (pristup kursu
  `prod_0OiixT2UwJbFm`) → generiše unique checkout/access link → sklapa **email_html** sa pristupom
  + bonusima → vraća `{checkout_url, plan_id, email_html, bonus_type}`.
- **Fast/regular split:** **vremenski** — `bonus_type = (uplata u 20:00–21:34 Beograd) ? 'fast' : 'regular'`.
  Email uslovno ubacuje "BRZI BONUSI" sekciju samo ako je `fast`. Code ima i hook za eksplicitan
  `bonus_flag` (preko `inputData.bonus_flag`) — ako se kasnije doda `client_reference_id` na Stripe CTA.
- **Stripe linkovi (2):** `4gMfZh0fH0FEgd9dR6asg0s` (start/snimka) i `4gM9ATe6x2NM4urcN2asg0v`
  (ponuda) — oba 97.99. Zapier se okida na BILO KOJU subscription, filtrira po ceni (nije vezan za link).

**⚠️ Whop API key** je hardkodovan u Zapier Code koraku (`Bearer apik_...`). **TREBA ROTIRATI**
(bio izložen). NIJE u ovom repo-u.

---

## 5. ✅ ŠTA JE URAĐENO (verifikovano)

**Stranice:**
- Dinamični countdown po `?datum=` na sve 3 strane
- TZ fix (`belgradeEpoch`, CEST/CET)
- /start: brzi bonusi prozor 20:00–21:34, pa redirect na `?datum=`
- Param → 4-dnevna ponuda (datum+4 @ 23:59:59)
- Sticky datum (localStorage), re-optin handling
- Expired = "istekla" siva (popup + optin gate **ISKLJUČENI** na zahtev klijenta)
- Syntax svih fajlova OK; logika Node-testirana (svi scenariji)
- Sve 3 paste-ovane na GHL, na pravim URL-ovima, plave, rade

**Backend (Zapier):**
- Code modifikovan: vremenski fast/regular split, `bonus_type`, uslovni BRZI BONUSI u mejlu
- input `created` mapiran na Stripe "Created"; `bonusType` linija = produkciona (ne hardkod 'fast')
- Zap UKLJUČEN, uspešni run-ovi potvrđeni, oba Stripe linka = 97.99

---

## 6. 🟡 ŠTA JOŠ TREBA (TODO — sledeća faza, follow-up posle webinara)

> **PODELA POSLA (KO RADI ŠTA):** Ovaj handoff je za osobu koja **PREUZIMA projekat**.
> - **Mejlove + WhatsApp poruke (tekstovi + `?datum=` linkovi) radi DUŠAN** (sekcija 6.2).
> - **SVE OSTALO radi osoba koja čita ovaj doc** — Hyros (6.3, prioritet), VSL video swap,
>   testiranje cele sekvence, security cleanup (6.5). Funnel je dignut na brzinu — proći SVE pedantno.
>
> **STATUS UPDATE (30.06.2026):**
> - **6.1 Make mapiranje datuma → ✅ URAĐENO** (`webinar_date` → custom field `evergreen_webinar_datum`).
> - **6.2 Mejl/WA tekstovi → verovatno radi DUŠAN** (samo paziti na format linka, vidi 6.2).
> - **6.3 Hyros parametri → 🔴 PRIORITET** (može biti prvo što se radi).
> - **6.4 snimka gornji countdown → ✅ ODLUČENO: ističe ISTO kao ponuda (datum+4)**, bez izmene.
> - **NOVO — VSL videi:** zameniti videe na `/snimka` i `/ponuda` (vturb player ID-evi u
>   `<vturb-smartplayer id="...">` + odgovarajući `players/<ID>/v4/player.js` preload i loader skripta).
> - **NOVO — „Zajedničko Polijetanje" brzi bonus UKLONJEN** sa `start.html` (frontend, jedini page koji
>   ga je imao; ostao „AI Zaposlenik (prvih 50)"). ⚠️ **Zapier mejl (BRZI BONUSI sekcija) JOŠ ima taj
>   bonus — ukloniti i tamo** za konzistentnost (inače fast kupci dobiju mejl sa bonusom kog nema na strani).
> - **Sve proveriti PEDANTNO sa timom** — funnel je dignut na brzinu; proći svaku kariku end-to-end.

### 6.1 Make: mapiranje datuma u custom field  ← **BLOKER za follow-up linkove**
- Custom field već napravljen: key **`evergreen_webinar_datum`** (Single line, na Contact).
- Make `webinar_date` Set Variable (format `DD_MM_YY`, npr `30_06_26`; posle 19:45 → sutra) treba
  da se **MAPIRA u taj custom field** u GHL **Update a Contact** i **Create + Update Contact** modulima.
- Bez ovoga, mejl/WA linkovi `?datum={{contact.evergreen_webinar_datum}}` idu **prazni**.
- (Preporuka: promeniti Make format na `DD_MM_YYYY` 4-cifre; parser i dalje vari oba.)

### 6.2 Mejl/WA tekstovi + linkovi
- Linkovi moraju biti: `https://uzivotrening.editunovac.com/start?datum={{contact.evergreen_webinar_datum}}`
  (NE `?{{datum}}`). Isto za `/snimka` i `/ponuda`.

### 6.3 Hyros parametri  ← **dogovoreno, NIJE implementirano**
Zahtev: parametri **moraju biti SVUDA, na svakoj strani i u svakom URL-u, nikad zagubljeni**.
- Dat after_live param: `?he={{contact.email}}&el=evergreen_webinar_after_live&htrafficsource=after_live&hgoal=evergreen_webinar&source=after_live`
- **Treba i LIVE varijanta** parametra (verovatno `_live` umesto `_after_live`) — pitati klijenta/tima.
- `he={{contact.email}}` je GHL merge tag — radi **samo u mejl/WA linkovima**. Na bare /start (koji
  čovek prekuca, nema kontakt) `he=` se NE može popuniti — samo statični deo (`el, htrafficsource, hgoal, source`).
- **Implementacija na stranicama:** trenutni redirect (`location.replace(... + "?datum=")`) **BRIŠE
  sve ostale parametre**. Treba prepraviti redirect da **ČUVA ne-`datum` parametre** (Hyros) i samo
  upravlja `datum`-om. Plus: dodati Hyros params na Stripe CTA href; i na bare /start u live prozoru.
  Guard protiv redirect petlje (ne redirektuj ako su parametri već tu).

### 6.4 snimka gornji countdown — ODLUKA
- "Snimka je aktivna još samo..." countdown trenutno koristi isti `offerEnd` (datum+4) kao ponuda.
- Odlučiti: da li replay ističe ranije od ponude (npr. datum+2)? Ako da → poseban anchor.

### 6.5 Security & cleanup
- **Rotirati Whop API key** (bio izložen; u Zapier Code koraku).
- (Opciono) re-enable popup + optin gate: vratiti `showExpiredPopup()` poziv u expired granu i optin
  redirect — funkcije su definisane ali isključene (`OPTIN_URL = .../optin`).
- (Opciono) migrirati Zapier → Make (klijent radije Make; uraditi NA MIRU sa testom, ne pod pritiskom).

---

## 7. Konvencije / zamke (PROČITATI pre editovanja)

- **GHL paste:** paste CEO fajl u Custom HTML/JS blok stranice. `.eun` kontejner je **full-bleed**
  (`width:100vw; margin-left:calc(50% - 50vw)`) pa sam prelama horizontalni padding GHL containera.
- **NE sudi po GHL builder preview-u** — JS (redirect/countdown) se tamo ponaša čudno. Testiraj na
  **objavljenoj** stranici (incognito).
- **`cleanTopBar` skripta** skriva GHL wrapper/topbar elemente i stray markdown tekst.
- **TZ:** uvek `belgradeEpoch`, NIKAD hardkod `+02:00`.
- **Encoding:** `snimka`/`repitch` JS stringovi ponegde koriste `\uXXXX` escape; `start` koristi raw
  UTF-8. Pri editovanju matchuj TAČNO kako stoji.
- **GHL slug konflikt:** svaki path je jedinstven po domenu; klonirane strane dobijaju auto-slug
  (`application-evergreen-...`). Ako `/snimka` servira pogrešnu stranu → druga strana/funnel/redirect
  drži taj slug; oslobodi ga (re-slug/obriši staro) pa publish.
- **Popup/optin gate su NAMERNO isključeni** (klijent: "neka stoji da je ponuda istekla"). Ne uključuj
  bez dogovora.

---

## 8. Kako testirati (countdown/param logika)

U **incognito** (Ctrl+Shift+N), zameni datume realnim (danas = npr. `30_06_2026`):

- **Param 4-dnevni:** `/start?datum=04_07_2030` → puno dana, ponuda aktivna, bez brzih bonusa, CTA aktivan.
- **Istek:** `/start?datum=20_06_2026` → "Ponuda istekla", CTA siv.
- **~2 dana:** `/start?datum=28_06_2026` → ~2 dana (do 02.07).
- **Sticky:** otvori `/start?datum=01_01_2030` → pa čist `/start` → redirect na `?datum=01_01_2030`.
- **Prelaz 21:34:** ostavi bare `/start` otvoren i gledaj u 21:34 → sam se prebaci na 4-dnevni.
  (Ili pomeri PC sat na 21:40 → reload bare `/start` → redirect na `?datum=danas`. Vrati sat.)

Lokalno (bez GHL-a): `node` za proveru date-matematike; svi `<script>` blokovi prolaze `new Function()` syntax check.

---

## 9. Eksterni sistemi i pristup

| Sistem | Šta drži | Napomena |
|---|---|---|
| **GHL** | 3 stranice (funnel "01 - Evergreen Webinar"), domen, custom field `evergreen_webinar_datum`, optin/thankyou | Custom HTML/JS blokovi |
| **Stripe** | 2 payment linka (oba 97.99), subscription | `4gMfZh...` (start/snimka), `4gM9AT...` (ponuda) |
| **Whop** | isporuka pristupa (`prod_0OiixT2UwJbFm`), free 1-use plan po kupcu | API key u Zapier-u (ROTIRATI) |
| **Zapier** | fulfillment Zap (Stripe→Filter→Code→Gmail) | NIJE u repo-u; razmotriti migraciju u Make |
| **Make** | `webinar_date` Set Variable, optin webhook, replay-gate webhook (`hook.eu1.make.com/...`) | mapiranje u custom field = TODO |
| **Meta pixel** | `697410579446946` | u stranicama (advanced matching iz URL ?email/?phone) |
| **Hyros** | tracking | NIJE postavljen; params = TODO |

---

## 10. Brzi start za novog (i Claude-a)

1. Pročitaj sekcije 3 (countdown) i 4 (backend) da razumeš tok.
2. Glavni preostali posao = sekcija 6: **Make mapiranje (6.1), mejl tekstovi (6.2), Hyros (6.3)**.
3. Pre bilo koje izmene stranica → pročitaj sekciju 7 (zamke).
4. Posle izmene → testiraj po sekciji 8, pa paste na GHL i testiraj objavljeno (incognito).
5. Sve karike za LIVE rade; preostalo je **follow-up** (dani posle webinara) + Hyros + security cleanup.
