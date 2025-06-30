# stremio-selfhosted  
**Stremio Stack con Mammamia, Media Flow Proxy e altro ancora**

Questo repository contiene istruzioni, configurazioni e suggerimenti per il self-hosting sul proprio NAS domestico di un'intera istanza privata di Stremio, con plugin **Mammamia**, **media-flow-proxy**, **StreamV** e altri componenti opzionali.

---

## 📢 Disclaimer  

> Questo progetto è a scopo puramente educativo.  
> L'utilizzo improprio di componenti che accedono a contenuti protetti da copyright potrebbe violare le leggi del tuo paese.  
> **Usa questi strumenti solo per contenuti legalmente ottenuti.**  
> L’autore non si assume responsabilità per eventuali usi illeciti.

---

## ✅ Requisiti  

Hai deciso di configurare una tua istanza privata di Mammamia e Media Flow Proxy, senza spendere un centesimo? Ecco cosa ti serve:

### 🖥️ Hardware e sistema operativo
- Un **PC o NAS** collegato alla rete locale.
- Una distribuzione Linux basata su Debian (es. Ubuntu).  
  > *(Le istruzioni sono per Ubuntu, ma facilmente adattabili ad altre distro.)*

### 🌍 Accesso remoto con IP dinamico + Port Forwarding

Poiché molti provider Internet assegnano un **IP pubblico dinamico**, è necessario un sistema per mantenere accessibile il tuo server anche quando l’IP cambia.
- Un **IP pubblico** (va bene anche se dinamico).
- Un account gratuito su [**No-IP.com**](https://www.noip.com/) per creare **hostname statici** che puntano sempre al tuo NAS.
- Il tuo router deve eseguire un **Port Forwarding**:
  - Porta **80** (HTTP) → verso la **porta 8080** del tuo NAS
  - Porta **443** (HTTPS) → verso la **porta 8433** del tuo NAS

> 🔁 Questo setup è fondamentale per permettere a Nginx Proxy Manager di ottenere e rinnovare automaticamente i certificati SSL tramite Let’s Encrypt.


### 🔐 Creazione degli hostname su No-IP

Per accedere alle tue applicazioni da remoto, devi creare 3 hostname pubblici gratuiti su [No-IP.com](https://www.noip.com/).

> ⚠️ Gli hostname devono essere univoci. Il mio consiglio è quello di aggiungere un identificativo personale (es. il tuo nome o una sigla) per evitare conflitti.

#### Esempi di hostname personalizzati:
- `mammamia-mario.ddns.net`
- `mfp-mario.ddns.net`
- `streamv-mario.ddns.net`
Puoi ovviamente scegliere qualsiasi nome, purché sia disponibile e facile da ricordare.

Questi hostname punteranno sempre al tuo NAS anche se il tuo IP cambia.  
Il tutto è possibile installando un piccolo agente (Dynamic DNS client) che aggiorna automaticamente il record DNS.

### 🍴 Consigliato: fai un fork del repository

> ✨ E' consigliabile creare un **fork personale** di questo repository su GitHub, in modo da poterlo modificare facilmente secondo le tue esigenze.
> Per farlo ti servirà anche un account su GitHub

Per fare ciò:
1. Vai sulla pagina del repository originale
2. Clicca su **"Fork"** (in alto a destra)
3. Clona il tuo fork sul NAS:

```bash
git clone https://github.com/<il-tuo-utente>/<nome-repo>.git
cd <nome-repo>
```
---

## 🔧 Componenti del progetto

| Servizio           | Porta interna | Descrizione                              |
|--------------------|---------------|------------------------------------------|
| **Mammamia**       | 5000          | Plugin personalizzato per Stremio        |
| **Media Flow Proxy (MFP)** | 3000   | Proxy per streaming video                |
| **StreamV**        | 4000          | Web player personalizzato (opzionale)    |
| **Nginx Proxy Manager** | 80/443/81 | Reverse proxy + certificati Let's Encrypt |
| **No-IP DUC (Docker)** | —         | Aggiorna il DNS dinamicamente            |

---

## 🔧 Configurazione hostname statici con No-IP

Se il tuo IP pubblico è dinamico, No-IP ti permette di associare un hostname che si aggiorna automaticamente ogni volta che il tuo IP cambia. Ecco come fare:

---

### 1. Registrazione e login

- Vai su [https://www.noip.com/](https://www.noip.com/)
- Clicca su **Sign Up** e crea un account gratuito.
- Verifica la tua email e accedi con le credenziali create.

### 2. Creazione degli hostname (Dynamic DNS)

- Dopo il login, clicca su **Dashboard** → **DDNS & Remote Access** → **No-Ip Hostnames** → **Create Hostname**.
- Inserisci il nome host, ad esempio:

  - `mammamia-mario e selezionate un dominio come ad esempio .ddns.net`
  
  Scegli un nome unico che ti permetta di riconoscerlo facilmente.

- Nel campo **Record Type** lascia selezionato **DNS Host (A)**.
- Nel campo **IPv4 Address**, vedrai il tuo IP pubblico attuale (se errato, correggilo con quello giusto).
- Premi **Create Hostname**.
<img width="1811" alt="Screenshot 2025-06-28 at 19 15 26" src="https://github.com/user-attachments/assets/437f32d3-7db1-40b4-b864-4fa33a072625" />

### 3. Ripeti per gli altri due hostname

- Crea altri due hostname per:

  - `mfp-mario.ddns.net`
  - `streamv-mario.ddns.net`
<img width="1811" alt="Screenshot 2025-06-28 at 19 20 04" src="https://github.com/user-attachments/assets/d432cfdb-7763-4f48-8f92-51622acd4f16" />

> ⚠️ **Attenzione:** Se vedi una scritta gialla accanto a un hostname su No-IP, significa che **l'aggiornamento automatico dell'indirizzo IP non è attivo**.  
> Andremo successivamente a configurare correttamente il client No-IP (DUC) per mantenerlo aggiornato.

> 📩 **Nota importante:** No-IP, nel piano gratuito, richiede il **rinnovo manuale mensile degli hostname**.  
> Riceverai un'email ogni 30 giorni per confermare **gratuitamente** che desideri mantenere attivo ciascun hostname.  
> Se **non li rinnovi**, gli hostname verranno disattivati e **non saranno più raggiungibili**.

---

## 📦 Docker + Docker Compose

> Questo progetto usa **Docker** per semplificare l’installazione e l’isolamento dei servizi.

### 📥 Installazione Docker

```bash
# 🔁 Rimuovi eventuali versioni precedenti
sudo apt remove docker docker-engine docker.io containerd runc

# 🔄 Aggiorna l’elenco dei pacchetti
sudo apt update

# 📦 Installa i pacchetti richiesti per aggiungere il repository Docker
sudo apt install -y ca-certificates curl gnupg lsb-release

# 🗝️ Aggiungi la chiave GPG ufficiale di Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 📥 Aggiungi il repository Docker alle fonti APT
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 🐳 Installa Docker, Docker Compose e altri componenti
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 👤 Aggiungi il tuo utente al gruppo docker
sudo usermod -aG docker $USER

# ⚠️ Per applicare il cambiamento, esegui il logout/login oppure:
newgrp

# ✅ Verifica che Docker funzioni correttamente
docker run hello-world
```

---

## 🚀 Avvio del progetto dal repository GitHub

Il progetto è contenuto in un repository GitHub che include un file `docker-compose.yml` preconfigurato. Alcuni servizi costruiranno automaticamente le immagini Docker a partire da Dockerfile remoti ospitati su GitHub.

### 🔧 Prerequisiti

Assicurati di avere:
- Docker e Docker Compose installati (vedi sezione precedente)
- Git installato (`sudo apt install git` se non lo hai)


### 📥 Clona il repository

```bash
cd ~
git clone https://github.com/tuo-utente/tuo-repo.git
cd tuo-repo
```

### 🏗️ Build delle immagini e avvio dei container

Per buildare le immagini (se definite tramite build: con URL GitHub) e avviare tutto in background:
```bash
docker compose up -d --build
```
> 🧱 Il flag --build forza Docker a scaricare i Dockerfile remoti ed eseguire la build, anche se l'immagine esiste già localmente.

### 🔍 Verifica che tutto sia partito correttamente

```bash
docker compose ps
```

Puoi anche consultare i log con:

```bash
docker compose logs -f
```

🔁 Aggiornare il repository e ricostruire tutto (quando aggiorni da GitHub)
```bash
git pull
docker compose down
docker compose up -d --build
```

---

## 🔐 Configurazione degli hostname e gestione SSL con Nginx Proxy Manager (NPM)

Per rendere accessibili le tue applicazioni web da internet in modo sicuro, useremo **Nginx Proxy Manager (NPM)**. Questo tool semplifica la gestione dei proxy inversi e automatizza l’ottenimento dei certificati SSL con Let’s Encrypt.

### 1. Creazione dei tre hostname su No-IP

Assicurati di aver creato 3 hostname statici su [No-IP.com](https://www.noip.com/) che puntino al tuo IP pubblico (anche se dinamico, aggiornato tramite l’agent No-IP):

- `mammamia-<tuo-id>.ddns.net`
- `mfp-<tuo-id>.ddns.net`
- `streamv-<tuo-id>.ddns.net`

> 🔔 **Suggerimento:** Usa un identificativo unico (`<tuo-id>`) per evitare conflitti con altri utenti No-IP.

### 2. Port Forwarding sul router

Per permettere il corretto funzionamento di NPM e il rinnovo automatico dei certificati SSL:

- Reindirizza la porta **80 (HTTP)** del router verso la porta **8080** del PC/NAS dove gira NPM.
- Reindirizza la porta **443 (HTTPS)** del router verso la porta **8443** del PC/NAS.

> Questo consente a Let’s Encrypt di verificare il dominio e rilasciare i certificati.

### 3. Configurazione dei proxy host in Nginx Proxy Manager

Per ogni applicazione, crea un nuovo **Proxy Host** in NPM seguendo questi passi:

- **Domain Names:** inserisci l’hostname corrispondente (es. `mammamia-<tuo-id>.ddns.net`)
- **Scheme:** `http`
- **Forward Hostname / IP:** l’indirizzo locale del NAS (es. `127.0.0.1` o IP locale)
- **Forward Port:** la porta interna dove l’app è in ascolto (es. `8000` per Mammamia)
- Abilita le seguenti opzioni:
  - **Block Common Exploits**
  - **Websockets Support** (se necessario)
  - **Enable HSTS** (opzionale, aumenta la sicurezza)
- **SSL tab:** seleziona:
  - **Enable SSL**
  - **Force SSL**
  - **HTTP/2 Support**
  - Spunta **Request a new SSL certificate from Let's Encrypt**
  - Accetta i Termini di servizio di Let’s Encrypt
  - Inserisci un indirizzo email valido per la registrazione SSL

Ripeti questa configurazione per ciascuno dei tre hostname con la rispettiva porta (ad esempio, `mfp-<tuo-id>.ddns.net` → porta `8001`, ecc.).

### 4. Verifica e manutenzione

- Dopo aver configurato i proxy host, prova ad accedere agli URL pubblici via browser.
- NPM gestirà automaticamente il rinnovo dei certificati SSL.
- Assicurati che il tuo router sia sempre configurato correttamente per il port forwarding, specialmente dopo eventuali riavvii o aggiornamenti firmware.

---


