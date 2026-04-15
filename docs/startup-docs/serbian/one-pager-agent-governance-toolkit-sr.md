## Product One-Pager: Agent Governance Toolkit (AGT)

- **Šta je to**: Open-source, MIT-licenciran **runtime sloj za upravljanje autonomnim AI agentima** (održava Microsoft, trenutno u fazi **Public Preview**) koji deterministički sprovodi politike nad **akcijama agenata** (pozivi alata, pristup resursima, poruke između agenata) *pre* izvršavanja.
- **Šta nije**: Proizvod za prompt guardrails / moderaciju sadržaja; upravlja onim što agenti *rade*, a ne onim što *govore*.

### Cilj
Omogućiti timovima da plasiraju autonomne agente u produkciju uz **determinističku, auditabilnu kontrolu** nad sposobnostima i ponašanjem agenata—bez značajnog povećanja latencije—tako da su bezbednosni i compliance zahtevi ispunjeni bez usporavanja isporuke.

### Definicija uspeha
- **Determinističko sprovođenje**: **0.00% stopa bezbednosnih kršenja** na objavljenom red-team benchmarku od 60 promptova (osnovna prompt-only bezbednost: **26.67% SVR**).
- **Zanemarljiv overhead**: Objavljeni benchmarkovi pokazuju **~0.011 ms** p50 za evaluaciju jedne politike, **~0.030 ms** p50 za evaluaciju 100 pravila, i **~0.103 ms** p50 za potpuno sprovođenje kernela po akciji.
- **Pokrivenost i usklađenost**: Demonstrirano mapiranje na **OWASP Top 10 za agentne aplikacije (2026)** sa **10/10 pokrivenih**, uz linkove ka kontrolama i komponentama.
- **Primenjivo u praksi**: Timovi mogu brzo integrisati upravljanje u postojeći agent framework (minuti, ne nedelje) koristeći adaptere/middleware, i mogu verifikovati stanje putem CLI-ja (`agt verify`, `agt doctor`).

### Pozadina (zašto ovo, zašto sada)
Agentni sistemi prelaze iz “chat” modela u **akciju**: pisanje fajlova, pozivanje API-ja, rad sa podacima i koordinaciju između više agenata. Dominantan bezbednosni obrazac—“recite modelu da prati pravila”—je inherentno probabilistički i može se zaobići putem jailbreak-a, socijalnog inženjeringa ili zloupotrebe alata.

AGT tretira akcije agenata kao syscalls: postavlja **tačku sprovođenja politike** između agent frameworka i sporednih efekata. Ovo je u skladu sa novim očekivanjima u 2026 (OWASP Agentic Top 10, NIST AI RMF, predstojeća regulatorna kontrola) gde organizacije zahtevaju **audit tragove, princip najmanjih privilegija i runtime kontrole**—ne samo best-effort promptove.

### Obavezne stavke (opseg v1.0)
- **Presretanje akcija + deterministički policy engine** za pozive alata/pristup resursima/poruke sa allow/deny odlukama pre izvršavanja.
- **Policy-as-code** koji timovi mogu pregledati i isporučiti (podržava YAML i uobičajene policy ekosisteme kao što su OPA/Rego i Cedar).
- **Auditabilnost**: Strukturisani audit logovi pokušaja akcija i odluka; izvoz u standardne observability pipeline-ove.
- **Framework-agnostičke integracije** (middleware/adapteri) tako da radi sa postojećim stackovima bez potrebe za novim frameworkom.
- **Verifikacioni UX**: CLI koji može da “doctor/verify/lint” stanje upravljanja za CI/CD i izveštavanje zainteresovanim stranama.
- **Dokumentovane bezbednosne granice** i preporučeni **defense-in-depth** model implementacije (upravljanje + izolacija kontejnera/infrastrukture).

### Stavke van opsega (eksplicitno nisu uključene)
- **Bezbednost modela / moderacija sadržaja** (koristiti poseban sloj kao što su filteri sadržaja/guardrails za izlaze).
- **Izolacija OS kernela ili hardvera** (AGT je na aplikacionom sloju; koristiti kontejnere/VM-ove za OS-level izolaciju).
- **Ispravnost rezonovanja**: AGT ne određuje da li je dozvoljena akcija “pametna”, tačna ili bez halucinacija.
- **Verifikacija ishoda**: Audit logovi beleže pokušaje/odluke; ne garantuju uspeh u spoljašnjem svetu.
- **Kontrola namere na nivou workflow-a (za sada)**: Pojedinačno dozvoljene akcije mogu se kombinovati u štetne workflow-e; ovo je poznato ograničenje sa mitigacijama i planiranim unapređenjima.

### Konkurencija / alternative
- **Prompt guardrails + “safety promptovi”**: Lako za početak, ali probabilistički i podložni zaobilaženju; nude slabe garancije za bezbednost akcija.
- **Alati za moderaciju sadržaja**: Korisni za filtriranje izlaza, ali ne sprečavaju agenta da izvrši nebezbedne akcije.
- **Generički policy engine-i (npr. OPA)**: Snažan gradivni blok, ali obično zahtevaju prilagođeno povezivanje sa agent frameworkovima i ne pružaju kompletan end-to-end stack za upravljanje agentima (identitet/poverenje/audit/adapteri).
- **Samo sandboxing (kontejneri/gVisor/Kata)**: Pomaže u izolaciji, ali ne daje granularnu, policy-driven kontrolu nad tim *koje* akcije su dozvoljene niti pruža audit semantiku na nivou upravljanja.

### Ključni vremenski elementi
- **Odmah (pilot)**: Pokrenuti proof-of-concept obavijanjem jednog postojećeg agent workflow-a AGT politikama, meriti blokirane akcije + kvalitet audita, i validirati uticaj na latenciju (očekivano zanemarljiv u odnosu na LLM pozive).
- **Kratkoročno (utvrđivanje)**: Proširiti politike/šablone za alate i tokove podataka sa najvećim rizikom; povezati audit događaje sa logging/SIEM sistemima i definisati SLO-ove za pouzdanost agenata.
- **Pre šire implementacije**: Odrediti potreban nivo izolacije (kontejneri/VM-ovi po agentu, mrežne politike) i validirati compliance mapiranja relevantna za vašu domenu.
- **Rizik za planiranje**: AGT je u fazi **Public Preview** i može uvoditi breaking promene pre GA; ranu adopciju tretirati kao iterativnu integraciju, a ne jednokratnu instalaciju.

### Reference (repo)
- `README.md` (pregled + mogućnosti + instalacija)
- `QUICKSTART.md` (10-minutni vodič za integraciju)
- `docs/OWASP-COMPLIANCE.md` (mapiranje na OWASP Agentic Top 10)
- `BENCHMARKS.md` + `packages/agent-os/modules/control-plane/benchmark/README.md` (metodologija benchmarka latencije/protoka i bezbednosnih kršenja)
- `docs/LIMITATIONS.md` (granice dizajna i mitigacije)