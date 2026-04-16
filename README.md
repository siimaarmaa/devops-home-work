# Spring PetClinic – Ansible proovitöö

See on minu proovitöö, kus ma panen Spring PetClinic rakenduse Ubuntu 24.04 serverisse käima ühe Ansible playbookiga. Eesmärk oli, et ma ei peaks serveris midagi käsitsi tegema — kõik seadistused tulevad koodist.

## Mida ma tegin

- Ansible playbook valmistab serveri ette (paigaldab Docker, Java, tulemüür jne)
- Kloonib PetClinic GitHubist ja ehitab selle Maveniga
- Paneb kõik konteineritesse (backend, PostgreSQL, nginx)
- Nginx termineerib TLS-i (sert on self-signed, sest see on testimiseks)
- Backend ja andmebaas on erinevates võrkudes
- GitHub Actions teeb deploy automaatseks iga push'i peale

## Kuidas seda käivitada

Vaja on:
- Ubuntu 24.04 server (nt Hetzner, DigitalOcean, Proxmox VM)
- SSH RSA-võti lokaalses masinas (`~/.ssh/id_rsa`, paroolikaitsega või ilma), mis on serveri `ubuntu` kasutaja juures lubatud
- Ansible 2.15+ lokaalses masinas

```bash
# 1. Pane oma serveri IP inventory faili
sed -i 's/YOUR_SERVER_IP/1.2.3.4/' ansible/inventory.ini

# 2. Kui võti on paroolikaitsega, lisa see ssh-agenti
ssh-add ~/.ssh/id_rsa
# (küsib parooli üks kord, jätab meelde kuni agent jookseb)

# 3. Käivita
cd ansible
ansible-playbook playbook.yml
```

**Teise võtme kasutamine:** kui su avalik võti pole `~/.ssh/id_rsa.pub`, anna tee:
```bash
SSH_PUBKEY_PATH=~/.ssh/id_ed25519.pub ansible-playbook playbook.yml
```

Kui kõik läks läbi, ava brauseris `https://<serveri-ip>/`. Brauser hoiatab sertifikaadi pärast (sest see on self-signed) — kliki "Advanced" ja mine edasi.

---

## Arhitektuur

### Komponendid

| Komponent | Mis see on | Mida teeb |
|---|---|---|
| nginx | Reverse proxy (alpine image) | Võtab vastu HTTPS-i 443 pordis, saadab backendile edasi |
| backend | Spring Boot 3 + Java 17 | PetClinic rakendus, kasutab `postgres` profiili |
| db | PostgreSQL 16 | Hoiab andmed alles |

### Võrgud

Tegin kaks erinevat Docker võrku, et komponendid oleks üksteisest eraldatud:

```
Internet ──443──► nginx ──────► backend ──────► postgres
                   (frontend võrk)     (backend võrk, internal)
```

- **`frontend` võrk** – siin on nginx ja backend. Nginx saab backendile ligi.
- **`backend` võrk** – siin on backend ja db. See võrk on märgitud `internal: true`, mis tähendab et sinna ei pääse internetist ligi ja db ise ei saa internetti minna.
- Nginx **ei ole** backend võrgus, seega ta ei saa otse db-ga rääkida. Peab läbi backendi minema.
- Db ei ole hostis ühtegi porti avanud (`expose:` ainult, mis on Docker-sisene).

Nõuded:
-  Internet → Proxy (port 443 on lahti)
-  Proxy → Backend (samas frontend võrgus)
-  Backend → DB (samas backend võrgus)
-  Internet → Backend (ei mapi porti, ei ole frontend võrgu väljapääs)
-  Internet → DB (`internal: true`)
-  Proxy → DB otse (nginx pole backend võrgus)

---

## Miks ma valisin need tehnoloogiad

**Ansible** – sest ülesanne nõudis. Aga ka ilma selleta oleks ma selle valinud, sest ei pea serverisse mingit agenti paigaldama, ainult SSH ligipääs. Alternatiiv oleks olnud Terraform, aga see on pigem infra loomiseks (VM-id, võrgud), mitte sees seadistamiseks.

**Docker Compose** – lihtsaim variant ühe serveri jaoks. Swarm oleks üle võlli (ei ole mitut masinat), Kubernetes oleks veel suurem ülepakkumine. Compose on ka Compose võrkudega (`internal: true` jne) päris heasti segmenteeritav.

**NGINX** – Traefik oleks ka töötanud ja automaatsed Let's Encrypt sertifikaadid oleks tore, kuna nginx lihtsam aru saada — üks config fail, loogika on lineaarne.

**PostgreSQL** – sest PetClinicus on olemas `postgres` profiil.

**Build hostil, mitte multi-stage Dockerfile** – ehitan jar-i otse serveris Maveniga ja siis kopin selle Docker build-konteksti. Multi-stage oleks puhtam, aga esimene build oleks kordades aeglasem (Docker peaks iga kord kõik sõltuvused uuesti alla laadima). Praegune variant cacheb `.m2/` kataloogi.

---

## Turvalisus

### Mis on hästi

- **SSH ainult võtmega** — `PasswordAuthentication no`, `PermitRootLogin no` (eraldi drop-in fail `sshd_config.d/` kataloogis, et mitte lõhkuda vaikekonfiguratsiooni)
- UFW tulemüür laseb sisse ainult portid 22, 80, 443
- fail2ban kaitseb SSH brute-force vastu
- HTTP suunab automaatselt HTTPS-ile (301 redirect nginxis)
- Backend jookseb konteineris mitte-root kasutajana (`spring`)
- Andmebaas pole hostis avatud, ainult Docker-sisene
- Eraldi süsteemikasutaja `petclinic` (mitte root) rakendusele

### Mida saaks paremini teha

- Self-signed sert võiks olla Let's Encrypt — vajaks domeeni nime ja certbot'i (või Traefik + ACME, mis teeks seda automaatselt)
- Postgres parool on praegu compose-failis. Parem oleks Docker secrets või keskkonnamuutujad `.env` failist (mis pole repos)
- Puudub container image scanning (Trivy vms)
- Pole unattended-upgrades automaatseks security patch'imiseks
- DB backup'e ei tee keegi — tootmises oleks vaja cron + pg_dump mõnda objektimälusse
- Logid lähevad ainult Docker logiseisu, pole keskselt kogutud

---

## Kuidas see liiguks Kubernetesesse

Kui selle lahenduse tootmises Kubernetesesse viia, siis:

| Praegu | Kubernetesis |
|---|---|
| Docker Compose service | `Deployment` + `Service` |
| `frontend` võrk | `NetworkPolicy` mis lubab nginx → backend |
| `backend` võrk (internal) | `NetworkPolicy` mis lubab backend → db ja keelab db egressi |
| nginx + self-signed | `Ingress` (nginx-ingress controller) + cert-manager + Let's Encrypt |
| Postgres konteiner | `StatefulSet` + `PersistentVolumeClaim`, või veel parem — managed DB (RDS, Cloud SQL) |
| Compose env vars | `ConfigMap` tavaliste jaoks + `Secret` paroolide jaoks |
| Ansible playbook serveri seadistamiseks | Läheb ära — klaster on juba olemas. Jääb ainult Helm chart rakenduse paigaldamiseks |
| Manuaalne deploy | CI ehitab image, lükkab registry'sse, ArgoCD/ sünkroniseerib klastrisse (GitOps) |
| Hosti UFW | Cloud firewall + `NetworkPolicy` podide vahel |

Kõige suurem muutus oleks ma arvan **andmebaas** — Kubernetes ei ole kunagi päris hea koht stateful asjade jaoks, seega selle annaks pilveteenusele. Ja **võrgu isolatsioon** läheb teistsuguseks — Docker bridge võrkude asemel on namespace-id + NetworkPolicy-d, mis on võimsam aga ka keerulisem aru saada.

---

## CI/CD – GitHub Actions

`.github/workflows/deploy.yml` fail käivitab Ansible playbooki automaatselt iga kord kui `main` harusse midagi push'itakse. Saab ka käsitsi GitHub UI-st käivitada (`workflow_dispatch`).

### Mida tuleb repos seadistada

Mine **Settings → Secrets and variables → Actions → New repository secret** ja lisa need neli saladust:

| Secret | Mis see on |
|---|---|
| `SSH_PRIVATE_KEY` | Privaatvõti sisu — terve `~/.ssh/id_rsa` fail koos `BEGIN`/`END` ridadega |
| `SSH_PUBLIC_KEY` | Avalik võti — `~/.ssh/id_rsa.pub` sisu |
| `SSH_PASSPHRASE` | Privaatvõtme parool (kui võti on paroolikaitsega) |
| `SERVER_HOST` | Serveri IP või domeen |
| `SERVER_USER` | Sisselogimiskasutaja serveris (nt `ubuntu`) |

Hetzneri automaattestiks on vaja lisaks:

| Secret | Mis see on |
|---|---|
| `HCLOUD_TOKEN` | Hetzner Cloud API token (Console → Security → API tokens, Read & Write) |

### Mida workflow teeb

1. Kloonib repo GitHub runnerisse
2. Paigaldab Ansible ja rsync
3. Kirjutab SSH võtmed `~/.ssh/` alla — nii privaat- kui avaliku (vaja mõlemat, sest `02-user.yml` loeb pubkey-d kohalikust failist et panna see serveri `authorized_keys` sisse)
4. Lisab serveri hosti `known_hosts`-i `ssh-keyscan` abil
5. Genereerib `inventory.ini` secretitest (nii ei ole serveri IP repos sees)
6. Käivitab `ansible-playbook playbook.yml`

Võib seda iga push'iga rahus käivitada — midagi ei lõhu. Ainult muutunud osa (nt uus jar) build'itakse uuesti.

### Kuidas ma seda turvalisemaks teeks

- See SSH võti peaks olema **eraldi deploy-key**, mitte sinu isiklik võti
- `ssh-keyscan` on ok aga mitte ideaalne — parem oleks hosti key `known_hosts`-i sisu ka secretina hoida, et MITM-i rünnak esimesel korral ei õnnestuks
- Võiks lisada GitHub Environment'i (nt `production`) ja panna sinna required reviewer, et iga push ei deployks automaatselt
- Võiks eraldada build ja deploy kaheks jobiks ja hoida jar-i artefaktina

---

## Failistruktuur

```
petclinic-ansible/
├── .github/workflows/
│   └── deploy.yml            # CI/CD — ansible-playbook iga push'i peale
│   └── test-hetzner.yml      # Ajutine test Hetzneris (loo VM + test + kustuta)
├── ansible/
│   ├── ansible.cfg           # Ansible üldsätted
│   ├── inventory.ini         # Serveri IP ja kasutaja
│   ├── playbook.yml          # Peamine fail — kutsub välja iga sammu eraldi failist
│   └── tasks/
│       ├── 01-base.yml       # apt update + java, maven, ufw, fail2ban
│       ├── 02-user.yml       # petclinic kasutaja + SSH võtme paigaldus
│       ├── 03-ssh.yml        # SSH karmistus — ainult võtmega
│       ├── 04-firewall.yml   # UFW — ainult 22/80/443 lahti
│       ├── 05-docker.yml     # Docker Engine + Compose plugin
│       ├── 06-build.yml      # git clone + mvn package
│       └── 07-deploy.yml     # TLS sert + docker compose up
├── docker/
│   ├── docker-compose.yml    # Kõik kolm teenust + kaks võrku
│   ├── backend/Dockerfile    # Spring Boot jar konteineris
│   └── nginx/
│       ├── Dockerfile
│       ├── nginx.conf        # TLS konfiguratsioon + proxy backendile
│       └── certs/            # Siia genereerib playbook self-signed serdi
├── README.md
└── TESTING.md                # Samm-sammult juhend Hetzner Cloudis testimiseks
```

## 

Playbooki saab ilma hirmuta mitu korda käivitada:
- `apt`, `user`, `ufw`, `blockinfile` — (ei tee midagi kui juba on tehtud)
- Maven build'il on `creates:` argument — ei käi uuesti, kui jar on juba olemas
- Sertifikaadi genereerimisel ka `creates:` — ei tee uut, kui olemas
- `docker compose up -d --build` ehitab ainult muutunud image'id uuesti

## Mida ma õppisin

- Ansible `import_tasks` vs `include_tasks` — valisin esimese, sest siis näeb `--list-tasks` kõik sammud ja saab `--start-at-task`-iga debugida
- Docker võrkude `internal: true` on päris cool — konteiner ei saa üldse internetti, aga teiste konteineritega samas võrgus saab suhelda
- `creates:` argument Ansibles on lihtsaim viis `command` mooduli idempotentseks saamiseks
- GitHub Actions secrets ei näita oma sisu logides isegi kui juhuslikult echo'd — see on tore
