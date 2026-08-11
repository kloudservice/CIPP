# CIPP Infrastruktur – Erkenntnisse & offene Punkte

## Azure Ressourcen (Stand: April 2026)

| Ressource | Name | Subscription |
|---|---|---|
| Resource Group | `CIPP` | Azure-Abonnement 1 |
| Static Web App | `cipp-swa-i56st` | |
| Function App (Haupt) | `cippi56st` | |
| Function App (Offload) | `cippi56st-2` | |
| App Service Plan (Haupt) | `CIPP-srv-i56st` | Y1 Consumption |
| App Service Plan (Offload) | `CIPP-srv-offload1` | Y1 Consumption |
| Storage Account | `cippstgi56st` | |
| App Insights | `cippi56st` | DefaultWorkspace WEU |

---

## Bekannte Probleme & offene TODOs

### 🐌 Cold Start – CIPP Login dauert 2-5 Minuten

**Problem:**  
Beide Function Apps laufen auf **Y1 Consumption Plan** (Dynamic). Azure skaliert diese auf
0 Instanzen wenn keine Requests für ~10-20 Minuten eingehen. Beim nächsten Login muss der
PowerShell-Worker neu starten und alle CIPP-Module laden → `Logging into CIPP` hängt
2-5 Minuten.

Gemessen am 2026-04-13: Cold Start > 180 Sekunden (kein Response innerhalb 3 Min Testfenster).

**Lösung (noch nicht umgesetzt):**  
Upgrade auf **Elastic Premium EP1** (~€150/Monat):
- 1x EP1 Plan für beide Function Apps (`cippi56st` + `cippi56st-2`)
- Always-Ready Instances = 1 → kein Cold Start
- `CIPP-srv-offload1` (Y1) kann danach gelöscht werden

**Schritte:**
```
1. Neuen EP1 Plan erstellen: CIPP-srv-ep1 (West Europe)
2. cippi56st auf EP1 Plan migrieren (ARM: serverFarmId ändern)
3. cippi56st-2 auf denselben EP1 Plan migrieren
4. WEBSITE_CONTENTAZUREFILECONNECTIONSTRING bleibt gleich
5. Alte Y1 Pläne (CIPP-srv-i56st, CIPP-srv-offload1) löschen
6. Im Azure Portal: Function App > Scale and concurrency > Always Ready = 1
```

---

### ✅ Repo-Umzug fkappen/CIPP → kloudservice/CIPP (2026-04-13)

**Was passiert war:**  
Beim Repo-Umzug wurde ein Workflow `main_cippi56st.yml` erstellt (Azure App Service
Auto-Deploy Wizard), der den Node.js-Frontend-Build auf die PowerShell Function App
`cippi56st` deployed hat – und damit die CIPP-API überschrieb. Resultat: 404-Fehler
auf allen API-Endpunkten.

**Behoben:**
- CIPP-API 10.3.0 (PowerShell) korrekt via Kudu ZIP-Deploy auf `cippi56st` deployed
- `main_cippi56st.yml` aus dem Repo gelöscht (Commit `41f0435e`)

---

### ✅ Function Offloading eingerichtet (2026-04-13)

`cippi56st-2` als Offload-Instanz deployed und konfiguriert.  
Die Instanz hat sich automatisch in der `Version` Storage Table registriert
(`cippi56st2*`-Tabellen vorhanden).

**Aktivierung:**  
In CIPP → Super Admin → Function Offloading prüfen ob beide Instanzen mit
Version `10.3.0` erscheinen, dann aktivieren.

---

### ℹ️ CIPP-API Deployment-Mechanismus

Die Function App `cippi56st` verwendet `WEBSITE_RUN_FROM_PACKAGE = 1` (lokales ZIP-Deploy
via Kudu). Updates erfolgen durch:

```powershell
# Neuestes ZIP von upstream laden, korrekt strukturieren und deployen
$zipUrl = "https://github.com/KelvinTegelaar/CIPP-API/archive/refs/heads/master.zip"
# 1. Herunterladen
# 2. Entpacken (innerer Ordner CIPP-API-master/)
# 3. Neu zippen mit host.json im Root
# 4. Via Kudu POST https://cippi56st.scm.azurewebsites.net/api/zipdeploy
```

Das `kloudservice/CIPP-API` Repository ist noch **nicht für Deployments konfiguriert**
(keine `AZURE_CONNECTION_STRING` Secret, Workflows haben Fork-Sperre).
Für automatische Updates: Secret hinterlegen und Fork-Bedingung aus `publish_release.yml` entfernen.

---

### ⚠️ Offload Function App `cippi56st-2` existiert nicht mehr (Stand 2026-08-11)

`cippi56st-2.azurewebsites.net` ist per DNS nicht mehr aufloesbar — die App wurde
offenbar geloescht. Der vom Azure-Wizard erzeugte Workflow `main_cippi56st-2.yml`
(deployte faelschlich das Frontend-Repo auf die Offload-API-App) schlug seit dem
13.04.2026 bei jedem Push fehl und wurde am 2026-08-11 aus dem Repo entfernt.

Falls Function Offloading wieder gewuenscht: neue Offload-App erstellen und mit
**CIPP-API**-Code deployen (nicht ueber dieses Frontend-Repo), danach in CIPP →
Super Admin → Function Offloading aktivieren. Vorher pruefen, ob in der `Version`
Storage Table noch alte `cippi56st2*`-Registrierungen haengen.