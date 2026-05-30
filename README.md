# openAut

> Öppen AI-plattform för fastighetsautomation med lokal inferens, Python edge-reglering och ingen leverantörsinlåsning.

openAut är ett **öppet tillägg till ditt befintliga BMS** — inte en ersättare. Plattformen samlar in driftsdata via MQTT over TLS, analyserar den lokalt med kraftfull AI-hårdvara, och kan driftsätta Python-baserad reglering direkt på edge-noder. Designat för offentlig upphandling, regelefterlevnad (ISO 27001, NIS2, IEC 62443, CRA, AI Act) och långsiktig förvaltning.

**Hemsida:** https://openaut.io · **Licens:** MIT · **Status:** v0.1 — aktiv utveckling

---

## Vad openAut gör

- **Feldetektering (FDD):** Korrelerar signaler över tid och identifierar rotorsak med konfidensgrad — levererat i Teams eller Slack, inte i BMS-klienten.
- **Energioptimering:** Prediktion, lastprognos och avvikelseanalys mot historisk trenddata.
- **Guidad integration:** Läs in en fabrikants manual, ange edge-nod — openAut guidar teknikern steg för steg och dokumenterar automatiskt.
- **Python edge-reglering:** Driftsätt Python-baserade reglerloopar direkt på edge-noden mot lokal I/O. Utan rundtur till AI-servern. Versionshanterat, loggat och återkallelsebart centralt via NemoClaw.
- **Automatisk dokumentation:** I/O-listor, MQTT topic-schema, driftsättningsprotokoll och FAT/SAT genereras som sidoeffekt av normal drift.

---

## Arkitektur — fyra lager

```
LAGER 04 — GRÄNSSNITT   Teams · Slack · Webb-HMI · REST API
LAGER 03 — AI           OpenClaw · NemoClaw · DGX Spark · Nemotron (lokal LLM) · FDD · Energioptimering
                         EMQX-broker · Telegraf (ingest) · TimescaleDB/PostgreSQL
LAGER 02 — EDGE         Linux-noder · SSH · Protokolldrivrutiner · Python edge-reglering
                         MQTT over TLS (stream + on-demand request/response via topics)
LAGER 01 — FÄLT         Modbus RTU/TCP · BACnet · M-Bus · LoRaWAN · KNX · DALI
```

**LAGER 01 — FÄLT:** Befintlig fältutrustning ansluts utan modifiering. PLC:er, DDC-regulatorer och mätare kommunicerar via sina befintliga fältprotokoll. openAut läser — fältets reglering behåller prioritet.

**LAGER 02 — EDGE:** Siemens SIMATIC IOT2050-noder kör standard Linux och nås av NemoClaw via krypterad SSH. Protokolldrivrutiner körs direkt på noden. NemoClaw kan via SSH även driftsätta Python-regleringsskript mot lokal I/O — slutna reglerloopar utan molnrundtur. Data transporteras krypterat till EMQX-brokern via MQTT over TLS — både som kontinuerlig mätström och som on-demand request/response via topics.

**LAGER 03 — AI:** NVIDIA DGX Spark (GB10 Grace Blackwell, 128 GB unified memory, 1 PFLOP FP4) kör EMQX-brokern, Telegraf (ingest till TimescaleDB/PostgreSQL) och agentstacken lokalt. Agentstacken är **NemoClaw** — NVIDIAs härdade referensimplementation ovanpå **OpenClaw** — som kör lokal **Nemotron**-inferens i en OpenShell-sandbox. NemoClaw läser historik och skriver analyser och larm till masterdatabasen. All data stannar i fastigheten.

**LAGER 04 — GRÄNSSNITT:** Insikter når de som behöver dem i de verktyg de redan använder. Drifttekniker i Teams. Energisamordnare i dashboard. Integrationsteam via REST API.

---

## Teknisk stack

| Lager | Komponenter |
|---|---|
| **openAut** (domänramverk) | BACnet-skill · Modbus-skill · M-Bus-skill · LoRa-skill · FDD-skill · Energianalys-skill · SSH edge-access · Python I/O-skill · Edge-reglerings-skill · MIT |
| **NemoClaw** (agentstack) | NVIDIAs referensimplementation ovanpå OpenClaw · OpenShell-sandbox (Landlock + seccomp + netns) · policy-baserade guardrails · lokal Nemotron-inferens · livscykelhantering · Apache 2.0 · alpha/tidig fas (mars 2026) |
| **OpenClaw** (agent-gateway) | Självhostad multi-channel-gateway för AI-agenter · Teams/Slack/m.fl. · skills & verktygsstöd · MIT · 250 000+ GitHub-stjärnor (mars 2026) |
| **Modell** (LLM) | NVIDIA Nemotron (öppna modeller) · lokal inferens · 200B+ lokalt, ~405B med två länkade DGX Spark |
| **AI-hårdvara** | NVIDIA DGX Spark (primär) · valfritt ARMv9-A-system ≥128 GB (Ampere · Qualcomm · Oracle · AWS Graviton-kompatibel) |
| **Edge-hårdvara** | Siemens SIMATIC IOT2050 |
| **I/O-modul** | Siemens EM1.8U (8× universell I/O · Modbus RTU · RS485) |
| **MQTT-broker** | EMQX · TLS · klientcertifikat · request/response-topics |
| **Ingest** | Telegraf (EMQX → TimescaleDB) |
| **Transport** | MQTT over TLS · WireGuard VPN |
| **Databas** | TimescaleDB · PostgreSQL · Haystack · Brick Schema |

> **Licenser i stacken:** openAut och OpenClaw är MIT-licensierade. NemoClaw är Apache 2.0 och i tidig förhandsfas (alpha, lanserad mars 2026) — openAut pinnar verifierade versioner och kapslar in stacken bakom stabila gränssnitt så att grunden kan utvecklas utan att driftmiljön påverkas.

---

## Kärnprinciper — vad openAut inte gör

**P.01 — Reglerar inte på autopilot**
openAut kan driftsätta Python-baserad reglering direkt på edge-noder via I/O, men aldrig på eget bevåg — bara när en applikationsprofil och ett explicit regleringsuppdrag konfigurerats av behörig användare. All skrivåtkomst loggas, versionshanteras och kan återkallas. Befintliga PLC:er och DDC-regulatorer behåller prioritet — openAut:s reglering är ett komplement, aldrig en override av säkerhetsfunktioner.

**P.02 — Skickar inte din data till molnet**
All inferens körs på den lokala AI-servern i fastigheten. Ingen driftdata lämnar byggnaden om du inte aktivt konfigurerar det.

**P.03 — Kräver inga proprietära beroenden**
Kärnstacken körs helt på öppen källkod: `pymodbus`, `BAC0`, `paho-mqtt`, EMQX, Telegraf, TimescaleDB med flera. Inga proprietära drivrutiner eller licenser.

**P.04 — Ersätter inte ditt befintliga BMS**
openAut samexisterar med Desigo CC, EcoStruxure, Trend, Regin, Niagara och alla andra plattformar. Det läser deras data — konkurrerar aldrig med dem.

**P.05 — Skapar inget nytt leverantörsberoende**
Arkitekturen är modulär, dokumenterad och kan tas vid av vilken kompetent integratör som helst. Varje designbeslut utvärderas mot frågan: skapar detta inlåsning?

---

## Referenshårdvara

### AI-lager
| Enhet | Chip | Minne | Prestanda | Roll |
|---|---|---|---|---|
| NVIDIA DGX Spark | GB10 Grace Blackwell | 128 GB unified | 1 PFLOP FP4 | Primär referenshårdvara |
| Valfritt ARMv9-A-system | Ampere · Qualcomm · Oracle · AWS Graviton-kompatibel | ≥128 GB | — | Alternativ — ingen hårdvarulåsning |

### Edge-lager
| Enhet | CPU | Gränssnitt | Roll |
|---|---|---|---|
| Siemens SIMATIC IOT2050 | TI AM6548 · 4× A53 · 1 GHz | RS232/422/485 · 2× GbE · Arduino Shield | Python-regulator och datainsamlare via SSH |
| Siemens EM1.8U | — | 8× universell I/O · Modbus RTU · RS485 | Industriell I/O, upp till 31 moduler per buss |

Hela stacken är ARM64-nativ. `pymodbus`, `paho-mqtt` och `BAC0` är verifierade på ARM64 utan proprietära beroenden.

---

## Säkerhet

Säkerhet är inbyggt från dag ett, inte tillagt i efterhand:

- VLAN-segmentering per lager
- MQTT over TLS med klientcertifikat för all datatransport från edge-noder
- WireGuard VPN för fjärråtkomst
- RBAC och auditloggning
- Regleringskod versionshanteras och granskas som konfiguration
- Separat säkerhetsagent på egen fysisk hårdvara med read-only SSH och listen-only Teams-bot — den observerar men kan inte agera, så en prompt injection saknar verkningsradie

### Regelverk — fem ramverk, tydlig ansvarsfördelning

openAut håller sig inom fem ramverk där vart och ett styr sin domän:

| Ramverk | Domän |
|---|---|
| **ISO 27001** | Ledningssystem och governance — grunden de övriga hängs upp i |
| **NIS2** | Verksamhetskrav och incidentstyrning (24h/72h-rapportering) |
| **IEC 62443** | OT/industriell cybersäkerhet — zonindelning, security levels, härdning |
| **CRA** | Cybersäker produkt med digitala komponenter — SBOM, sårbarhetshantering, uppdateringar |
| **AI Act** | AI-funktionens krav och styrning — mänsklig kontroll, transparens, loggning, dokumentation |

Se fullständig hotmodell, säkerhetsarkitektur och regelverksgenomgång: https://openaut.io/security.html

---

## Offentlig sektor och LOU

openAut är MIT-licensierat — det finns ingen enskild leverantör att upphandla. Plattformen är gratis att använda. Vad din organisation upphandlar är **kompetensen** att implementera och förvalta den — tjänster som kan tilldelas valfri regional konsult eller befintlig ramavtalspartner.

En lösning som en kommun bygger kan återanvändas av alla andra. Bidrag tillbaka till projektet stärker plattformen för hela ekosystemet.

---

## Fältprotokollstöd (edge → fält)

`Modbus RTU` · `Modbus TCP` · `BACnet/IP` · `BACnet MS/TP` · `M-Bus` · `LoRaWAN (EU868)` · `KNX` · `DALI`

**Transport edge → AI-lager:** `MQTT over TLS` via EMQX-broker (kontinuerlig ström och on-demand request/response)

---

## Kom igång

```bash
# Hemsida och dokumentation
https://openaut.io

# Systemtopologi — lager, protokoll, dataflöden och referenshårdvara
https://openaut.io/topologi.html

# IT/OT-säkerhet — hotmodell, säkerhetsarkitektur och compliance
https://openaut.io/security.html

# Källkod
https://github.com/openaut
```

Projektet är under aktiv utveckling och redo för pilotdriftsättningar. Bidra med en protokolldrivrutin, anpassa en applikationsprofil, eller testa edge-reglering mot din befintliga I/O-hårdvara.

---

MIT License · openAut · https://openaut.io
