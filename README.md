# Uppgift A – Kontaktformulär, Nordvind Bygg

Kontaktsida i HTML + automatiserat flöde i n8n som sorterar förfrågningar efter budget.

```
Kunden fyller i formuläret
        │  POST (JSON)
        ▼
   n8n Webhook ──► Normalisera ──► Giltig? ──nej──► 400 "kunde inte tas emot"
                                      │ ja
                                      ▼
                             Svara "Tack" direkt  ◄── sidan visar bekräftelsen
                                      │
                                      ▼
                          Logga i Google Sheets (alla)
                                      │
                                      ▼
                                 Budgetrutt
             ┌────────────────────────┼────────────────────────┐
     ≥ 150 000 kr              25 000–150 000 kr          < 25 000 kr
             ▼                        ▼                        ▼
   Mejl till Sara (sälj)     Mejl till gemensam        AI skriver svar
   märkt "🔴 STORT JOBB"         inkorg                 ──► mejl till kunden
```

## Filer

| Fil | Vad det är |
| --- | --- |
| `index.html` | Hela kontaktsidan – en fil, inga beroenden. Klistra in webhook-URL:en högst upp i `<script>`. |
| `n8n/nordvind-kontakt.json` | Workflow-export att importera i n8n. |
| `n8n/kalkylark-rubriker.csv` | Rubrikraden till kalkylarket (kolumnnamnen måste stämma exakt). |

---

## Del 1 – HTML-sidan

Sidan är klar. Det enda du behöver ändra är rad 279 i `index.html`:

```js
var WEBHOOK_URL = "https://DIN-N8N-INSTANS/webhook/nordvind-kontakt";
```

Så här fungerar den:

* **Fälten:** namn, e-post, telefon (frivilligt), typ av jobb, budget (tre val) och en beskrivning av jobbet. Beskrivningen är det språkmodellen använder för att skriva svaret.
* **Ingen omladdning:** `event.preventDefault()` stoppar den vanliga formulärpostningen. Datan skickas med `fetch()` och bekräftelsen visas genom att formuläret döljs och "Tack"-rutan visas.
* **Bekräftelsetexten kommer från n8n.** Svaret från webhooken (`rubrik`, `meddelande`, `referens`) skrivs ut på sidan, så kunden ser direkt vilken väg förfrågan tog och får ett ärendenummer.
* **Validering** sker både i webbläsaren (röda fält, inga alerts) och en gång till i n8n – en bott som postar direkt till webhooken kommer inte förbi.
* **Spamskydd:** ett dolt fält (`foretag`). Människor ser det aldrig, bottar fyller i det, och n8n märker förfrågan som spam och slänger den.
* **Felhantering:** om n8n inte svarar inom 20 sekunder visas en röd ruta med telefonnumret istället – formuläret ligger kvar ifyllt så kunden kan försöka igen.

### Kör sidan lokalt

Öppna **inte** filen med dubbelklick (`file://`) – då blockerar webbläsaren anropet till n8n. Starta en liten webbserver istället:

```bash
npx http-server . -p 3000     # eller "Live Server"-tillägget i VS Code
```

och gå till `http://localhost:3000`.

---

## Del 2 – Sätt upp n8n

### 1. Skaffa n8n

Enklast är **n8n Cloud** (gratis provperiod, `app.n8n.cloud`) eftersom webhooken då redan ligger på internet. Vill du köra lokalt går det lika bra:

```bash
npx n8n
# n8n startar på http://localhost:5678
```

Kör du lokalt måste HTML-sidan också köras lokalt (som ovan), annars når webbläsaren inte webhooken.

### 2. Importera flödet

I n8n: **Workflows → Add workflow → ⋯ (meny uppe till höger) → Import from File…** och välj `n8n/nordvind-kontakt.json`.

Du får 12 noder plus fyra gula anteckningar som förklarar stegen. Noderna med varningstriangel saknar inloggningsuppgifter – det fixar vi nu.

### 3. Koppla in konton (credentials)

| Nod | Konto | Så gör du |
| --- | --- | --- |
| `Mejla Sara (sälj)`, `Mejla teamet`, `Mejla kunden` | Gmail | Öppna noden → **Credential to connect with → Create new** → följ Googles inloggning. I n8n Cloud räcker "Sign in with Google". |
| `Logga i kalkylark` | Google Sheets | Samma sak, samma Google-konto. |
| `OpenAI-modell` | OpenAI | API-nyckel från `platform.openai.com/api-keys`. |

Alla tre mejlnoderna kan dela samma Gmail-credential – du skapar den en gång och väljer den i de andra två.

> **Inget Google-konto?** Byt ut de tre Gmail-noderna mot **Send Email** (SMTP) – samma fält, men du fyller i mejlserver, användarnamn och lösenord. Har du Gmail men vill slippa OAuth funkar SMTP med ett app-lösenord: `smtp.gmail.com`, port 465, SSL.

> **Ingen OpenAI-nyckel?** Ta bort `OpenAI-modell` och dra in **Google Gemini Chat Model** istället (gratis nivå). Koppla den till den lilla runda anslutningen under `AI skriver svaret`. Resten av flödet är oförändrat.

### 4. Skapa kalkylarket

1. Nytt kalkylark i Google Sheets, döp fliken till **Förfrågningar**.
2. Klistra in rubrikerna i rad 1 – exakt dessa, i den här ordningen:

   `Referens` · `Mottagen` · `Namn` · `E-post` · `Telefon` · `Typ av jobb` · `Budget` · `Budget min (kr)` · `Kategori` · `Beskrivning`

   (Finns färdiga i `n8n/kalkylark-rubriker.csv`.)
3. Öppna noden `Logga i kalkylark` i n8n:
   * **Document** → byt från "By ID" till **From list** och välj ditt ark (eller klistra in ark-ID:t ur adressfältet: `docs.google.com/spreadsheets/d/`**`DET_HÄR_ÄR_ID`**`/edit`).
   * **Sheet** → välj fliken `Förfrågningar`.
   * Under **Values to Send** ska de tio kolumnerna nu dyka upp med sina uttryck ifyllda. Stämmer något inte, klicka på fältet och dra in rätt värde från vänsterspalten.

### 5. Ändra mottagaradresser

Nordvind Bygg är ett påhittat företag och `nordvindbygg.se` tar inte emot mejl. Låter du adresserna stå kvar blir körningen ändå **grön** i n8n – Gmail sväljer adressen – och först några minuter senare kommer en studs i din inkorg. Byt därför innan du testar.

Båda interna adresserna sitter på **ett** ställe: öppna noden `Normalisera`, raderna längst upp:

```js
const MEJL_SALJ = 'sara@nordvindbygg.se';    // Sara på sälj – stora jobb
const MEJL_TEAM = 'offert@nordvindbygg.se';  // gemensam inkorg – mellanstora jobb
```

Skriv in adresser du faktiskt kommer åt. Har du bara en inkorg funkar Gmails plus-adressering utmärkt – allt hamnar hos dig, men du ser på mottagarraden vilken väg mejlet tog och kan filtrera på det:

```js
const MEJL_SALJ = 'dittnamn+sara@gmail.com';
const MEJL_TEAM = 'dittnamn+team@gmail.com';
```

`Mejla kunden` rör du inte – den skickar till adressen kunden fyllde i formuläret. När du testar det lilla jobbet fyller du alltså i din egen adress (gärna `dittnamn+kund@gmail.com`) i formuläret.

**Avsändare** blir det Gmail-konto du kopplade in, så mejlen kommer från din egen adress. Det är helt okej för redovisningen – vill du att det ska se skarpare ut kan du byta visningsnamn under Gmail → Inställningar → Konton → "Skicka e-post som".

### 6. Koppla ihop sidan med webhooken

n8n har **två** URL:er för samma webhook, och det är här det brukar gå fel:

| | URL | När den funkar |
| --- | --- | --- |
| Test | `…/webhook-test/nordvind-kontakt` | Bara medan du har klickat **Execute workflow** och n8n står och lyssnar. Ett anrop, sen slutar den lyssna. |
| Skarp | `…/webhook/nordvind-kontakt` | Alltid, men **bara när workflowet är Active** (reglaget uppe till höger). |

Öppna noden `Kontaktformulär`, kopiera den URL du vill använda och klistra in den i `index.html`:

```js
var WEBHOOK_URL = "https://ditt-konto.app.n8n.cloud/webhook/nordvind-kontakt";
```

CORS är redan förberett i noden (`Allowed Origins = *`), så sidan får läsa svaret oavsett var den ligger.

---

## Del 3 – Testa

Aktivera workflowet (reglaget **Active**), öppna sidan och skicka **tre** förfrågningar. Använd en riktig adress du kommer åt i alla tre – då ser du också AI-svaret. (Adresserna till Sara och teamet ska vara utbytta enligt steg 5.)

| # | Budget | Förväntat resultat |
| --- | --- | --- |
| 1 | Över 150 000 kr | Mejl till "Sara" med ämnesrad `🔴 STORT JOBB – …`. Sidan skriver: *"Din förfrågan har gått direkt till Sara…"* |
| 2 | 25 000–150 000 kr | Mejl till den gemensamma inkorgen. Sidan skriver: *"…återkommer inom ett arbetsdygn…"* |
| 3 | Under 25 000 kr | Kunden får ett mejl som språkmodellen skrivit utifrån beskrivningen. Sidan skriver: *"Du får ett mejl… inom några minuter."* |

Kontrollera sen:

* **Kalkylarket:** tre nya rader, en per förfrågan, med rätt värde i kolumnen `Kategori`.
* **Executions** (vänstermenyn i n8n): tre gröna körningar. Klicka in i en och du ser vilken gren som var aktiv – **det är den skärmdumpen du ska lämna in**. Ta den gärna på en körning där du öppnat flödesvyn så att den gröna vägen syns.

Skriv gärna beskrivningar med lite kött på benen i test 3 – det är hela poängen med AI-svaret. Till exempel: *"En altandörr som kärvar och två lister som lossnat i hallen. Huset är från 70-talet och dörren har hängt snett sen i våras."*

---

## Del 4 – Lämna in

- [ ] `n8n/nordvind-kontakt.json` – exportera din **egna** version igen när allt fungerar (**⋯ → Download**) så att dina justeringar kommer med.
- [ ] `index.html` – med din webhook-URL ifylld.
- [ ] Skärmdump från **Executions** som visar de tre körningarna.
- [ ] Bonus: skärmdump av kalkylarket med de tre raderna, och av AI-mejlet kunden fick.

---

## Felsökning

| Symptom | Orsak och lösning |
| --- | --- |
| Röd ruta: "Något gick fel när förfrågan skulle skickas" | Öppna webbläsarens konsol (F12). Står det **CORS** – kontrollera att `Allowed Origins (CORS)` är `*` i webhook-noden och att du inte öppnat sidan som `file://`. |
| `404 webhook not registered` | Workflowet är inte aktiverat (skarp URL), eller så har test-läget slutat lyssna. Klicka **Execute workflow** igen, eller aktivera workflowet och byt till `/webhook/`-URL:en. |
| Sidan visar "Tack" men inget hamnar i arket | Kolla **Executions** – körningen är röd på Sheets-noden. Oftast stavfel i en kolumnrubrik, eller fel flik vald. |
| Mejlen kommer inte fram | Kolla skräpposten. Kolla också att du bytt adresserna i `Normalisera` – `@nordvindbygg.se` finns inte och studsar. Ser credentialen gulmarkerad ut: öppna den och kör **Reconnect**. |
| Studsmejl: "Address not found" | Någon adress pekar fortfarande på den påhittade domänen `nordvindbygg.se`. |
| AI-noden är röd: "insufficient_quota" | OpenAI-kontot saknar saldo. Fyll på några dollar, eller byt till Google Gemini-noden enligt tipset i steg 3. |
| Fel budget-gren kördes | Öppna noden `Normalisera` i körningen och titta på `budgetMin`. Gränserna sätts i `Budgetrutt`: `≥ 150000`, `≥ 25000`, resten. |
| Tidsstämpeln i arket är en timme fel | `Mottagen` sparas i UTC. Vill du ha svensk tid: öppna `Logga i kalkylark`, ändra `Mottagen` till `{{ $now.setZone('Europe/Stockholm').toFormat('yyyy-MM-dd HH:mm') }}`. |

---

## Vill du visa upp lite extra?

* **Svara även mellanstora jobb med AI** – kopiera `AI skriver svaret` + `Mejla kunden`, koppla in dem på mellan-grenen och justera systemprompten.
* **Slack istället för mejl till Sara** – byt ut Gmail-noden mot en Slack-nod, samma uttryck i meddelandet.
* **Sammanfattning varje morgon** – lägg till ett **Schedule Trigger** + Google Sheets (read) + AI som sammanfattar gårdagens förfrågningar.
