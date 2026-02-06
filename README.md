# Automatiserad Guestbook-deployering med GitHub Actions

**Namn:** Thai Ha Dang. 
**Kurs:** CI/CD - DevOps 24. 
**Arkitektur:** Push-baserad deployering. 
**Plattform:** Red Hat OpenShift. 

---

## Översikt

Detta projekt implementerar och automatiserar livscykeln för en **3-lager (3-tier) Guestbook-app** som består av:

* **Frontend** (NGINX + HTML/JavaScript)
* **Backend** (Go REST-API)
* **Datalager** (PostgreSQL + Redis cache)

En **push-baserad CI/CD-pipeline** med **GitHub Actions** säkerställer att varje push till `main` automatiskt:

1. Bygger nya container-images
2. Pushar dem till GitHub Container Registry (GHCR)
3. Deployar den uppdaterade applikationen i ett OpenShift-kluster

Projektet visar **production-liknande arbetssätt** för CI/CD, containerisering och Kubernetes/OpenShift.

---

## Applikationsarkitektur

### 1. Frontend

* Single Page Application (`index.html`)
* Serveras med **NGINX**
* Visar gästboksinlägg
* Användare kan skapa nya inlägg
* Automatisk uppdatering var 30:e sekund
* Proxyar alla `/api/*`-anrop till backend

Frontend och backend är helt separerade komponenter och all kommunikation sker via backendens API.

### 2. Backend (API)

* Skriven i **Go**
* Exponerar ett REST-API med följande endpoints:

**GET /api/entries**

* Returnerar gästboksinlägg
* Läser från Redis-cache om möjligt (TTL: 30 sekunder)

**POST /api/entries**

* Skapar ett nytt gästboksinlägg
* Sparas permanent i PostgreSQL

**GET /api/stats**

* Returnerar statistik
* Totalt antal inlägg
* Cache-träffar och cache-missar

**/health**

* Health-check endpoint
* Kontrollerar anslutning till PostgreSQL och Redis
* Innehåller retry-logik (upp till 30 försök) vid start

### 3. Databas

* **PostgreSQL**

  * Permanent lagring
  * Körs som StatefulSet
  * Använder PersistentVolumeClaim (PVC)

* **Redis**

  * In-memory cache
  * Förbättrar prestanda och minskar belastning på databasen

---

## Containerisering

* Separata Dockerfiles för frontend och backend
* Använder RedHat UBI
* Backend använder multi-stage Dockerbuild:

  1. Kompilerar Go-binären
  2. Kopierar endast binären till en minimal runtime-image

Detta ger mindre, snabbare och säkrare images, eftersom containern bara innehåller det som behövs för att köra programmet. Extra filer och verktyg tas bort, vilket gör att containern startar snabbare och blir säkrare.

---

## Kubernetes / OpenShift-resurser

Alla resurser är definierade som kod i katalogen `k8s/`:

* **Deployments**

  * Frontend (2 repliker)
  * Backend (2 repliker)
* **StatefulSet**

  * PostgreSQL med persistent lagring
* **Deployment**

  * Redis
* **Services**

  * Intern kommunikation via DNS (`postgres`, `redis`, `backend`)
* **ConfigMap**

  * Icke-hemliga inställningar (hostnamn, portar)
* **Secrets**

  * Inloggningsuppgifter för databas och Redis
* **OpenShift Route**

  * Exponerar frontend publikt

---

## CI/CD-pipeline – GitHub Actions

Pipelinen definieras i `.github/workflows/pipeline.yml` och består av två huvudjobb.

### 1. Build (CI)

* **Trigger:** Push till `main`
* **Steg:**

  * Checkar ut koden
  * Bygger Docker-images för frontend och backend
  * Pushar images till **GitHub Container Registry (GHCR)**

---

### 2. Deploy (CD)

* **Beroende:** Körs endast om Build-jobbet lyckas
* **Steg:**

  * Installerar OpenShift CLI (`oc`) i GitHub-runnern
  * Loggar in i OpenShift med token lagrad i GitHub Secrets
  * Applicerar Kubernetes-manifest (`oc apply -f k8s/`)
  * Startar om deployments (`oc rollout restart`) för att tvinga pods att hämta nya images

---

## Konfiguration & Secrets

För att pipelinen ska fungera krävs följande **GitHub Actions-secrets**:

| Secret             | Beskrivning                                        |
| ------------------ | -------------------------------------------------- |
| `OPENSHIFT_SERVER` | OpenShift API-URL (t.ex. `https://api.sandbox...`) |
| `OPENSHIFT_TOKEN`  | Service Account-token för inloggning i klustret    |

---

## Problem & felsökning

### Databaspersistering (PVC)

* **Problem:** Uppdatering av `DB_PASSWORD` fungerade inte – PostgreSQL nekade inloggning.

Jag fastnade länge på att DB_PASSWORD inte uppdaterades, trots att jag ändrade mina Secrets. Jag insåg till slut att PostgreSQL bara sätter lösenordet första gången databasen skapas. Eftersom vår data sparades i en PVC, vägrade den byta lösenord vid omstart. Jag löste det genom att radera vår PVC och låta databasen initieras om från början. 

---

### OpenShift Route & port-namn

* **Problem:** Applikationen deplyars korrekt men kunde inte nås via publik URL.

Trots att allt såg grönt ut i OpenShift kunde jag inte nå appen via den publika URL:en. Jag upptäckte att OpenShift Routes ibland är kräsna med portnummer. Genom att byta ut portnumret mot ett namngivet interface (name: http i servicen och targetPort: http i routern) fick jag trafiken att flöda direkt.

---

## Repositoriets struktur

```text
├── .github/workflows
│   └── pipeline.yml       # CI/CD-pipeline
├── backend/               # Backend-kod (Go)
├── frontend/              # Frontend-kod (HTML/JS)
├── k8s/                   # Kubernetes / OpenShift-manifest
│   ├── route.yaml
│   ├── service.yaml
│   └── ...
└── README.md              # Projektdokumentation
```

## Bonusfrågor:

**Behöver vi apply:a alla YML-filer?**. 
Nej, egentligen inte, men vi kör oc apply -f k8s/ på hela mappen ändå. Det är enklast eftersom Kubernetes själv förstår vilka filer som har ändrats och bara uppdaterar det som behövs.

**Måste man starta om Deployment om man ändrar en ConfigMap?**.  
Ja. Poddarna läser bara in inställningarna när de startar. Om vi ändrar en ConfigMap fattar inte appen det förrän vi gör en oc rollout restart så att nya poddar skapas med de nya inställningarna.

**Hur fungerar privata containers?** Det har jag inte gjort.

## Ändringar:

Eftersom kursens OpenShift-kluster inte är uppe, valde jag att deploya Guestbook-applikationen på nytt i RH OpenShift Sandbox.

Först gjorde jag applikationens container-images privata i ett container-register. Efter det konfigurerade jag OpenShift så att klustret har tillgång till de privata images, genom att lägga till ett image pull secret genom den här kommand:

oc create secret docker-registry my-image-pull-secret \
  --docker-server=quay.io \
  --docker-username=hadang285 \
  --docker-password=********** \
  --docker-email=devops.engineer0528@gmail.com

Till sist uppdaterade jag deployment-filerna så att Guestbook-applikationen använder de privata images när den startas.

Länken till app: [Guestbook App](http://frontend-hadang285-dev.apps.rm1.0a51.p1.openshiftapps.com)