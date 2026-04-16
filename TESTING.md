# Testimine Hetzner Cloudis

See juhend aitab mul endal Hetzner Cloudis proovitöö käima panna ja läbi testida enne esitamist.

**Kaks varianti:**
- **Automaatne** (vt allpool "Otsetee: GitHub Actions teeb kõik ära") — GitHub Actions loob serveri, deployb, testib, kustutab. Üks klikk.
- **Käsitsi** — serveri loomine Console'is ja playbooki käivitamine enda masinast. Paremini näed mis toimub.

---

## Otsetee: GitHub Actions teeb kõik ära

`.github/workflows/test-hetzner.yml` workflow:
1. Loob Hetzneri API-ga uue CX22 serveri (nimi `petclinic-test-<run-id>`)
2. Paneb cloud-init'iga `ubuntu` kasutaja paika
3. Ootab SSH-d
4. Jookseb Ansible playbooki
5. Teeb testid:
   - `curl https://<ip>/` peab tagastama PetClinicu HTML-i
   - `http://<ip>/` peab andma 301 HTTPS-ile
   - `docker network inspect` peab näitama et backend võrk on `internal: true`
   - nginx ei tohi olla backend võrgus, db ei tohi olla frontend võrgus
6. Valikuliselt kustutab serveri

### Seadistus (ühekordne)

Lisaks olemasolevatele `SSH_PRIVATE_KEY`, `SSH_PUBLIC_KEY` secretitele on vaja:

| Secret | Kust saada |
|---|---|
| `HCLOUD_TOKEN` | Hetzner Console → Project → Security → API Tokens → Generate API token (Read & Write) |
| `HETZNER_SSH_KEY` | Nimi, millega sa oma SSH võtme Hetzneri Console'is lisasid. Kontrolli: Console → Security → SSH Keys → kopeeri võtme nimi |

### Käivitamine

GitHub → **Actions** → **Test on Hetzner** → **Run workflow**. Vali:
- **server_type**: `cx22` (piisav)
- **location**: `hel1` (Helsinki, lähim)
- **destroy_after**: 
  - `false` = jätab serveri alles (soovitatud, et saaksid ise ka uurida) — pärast testi käivita eraldi `Cleanup Hetzner test servers` workflow kui oled valmis
  - `true` = kustutab kohe pärast teste

Kogu jooks võtab aega ~10-15 minutit. Kui kõik sammud rohelised → töö on valmis esitama.

### Kui server jäi kogemata alles

Käivita `Cleanup Hetzner test servers` workflow — see kustutab kõik `petclinic-test-*` nimelised serverid. Või tee Console'is käsitsi.

---

## Käsitsi variant

## 1. Serveri loomine

Hetzner Cloud Console'is (https://console.hetzner.cloud):

1. **Projekti valik** — kas olemasolev või loo uus (nt "petclinic-test")
2. **Add Server**
3. Seaded:
   - **Location**: Helsinki (HEL1) — kõige lähemal, madalam latency
   - **Image**: Ubuntu 24.04
   - **Type**: CX22 (2 vCPU, 4 GB RAM) — PetClinicu build vajab vähemalt 2 GB, CX11-ga võib Maven out-of-memory visata
   - **Networking**: Public IPv4 lubatud (vaikimisi)
   - **SSH Keys**: lisa oma avalik võti (see sama, mida Ansible playbook kasutab — `~/.ssh/id_ed25519.pub`)
   - **Firewalls**: vahele jätta — meie Ansible playbook seadistab UFW niikuinii
   - **Name**: nt `petclinic-test`

4. Create & Buy Now (~€4.5/kuu, aga arve tehakse tunnipõhiselt — paar tundi testimist maksab mõned sendid)

Serverisse sisse logimise testimiseks:

```bash
ssh root@<serveri-ip>
```

Hetzner loob vaikimisi root kasutaja SSH võtmega. Meie playbook eeldab aga `ubuntu` kasutajat (nii on inventory.ini vaikeseades). Vaja on kas:

**Variant A** — muuta inventory, et see kasutaks `root`-i:
```bash
sed -i 's/ansible_user=ubuntu/ansible_user=root/' ansible/inventory.ini
```

**Variant B** — luua `ubuntu` kasutaja käsitsi esimesel ühendusel:
```bash
ssh root@<serveri-ip> '
  adduser --disabled-password --gecos "" ubuntu
  usermod -aG sudo ubuntu
  mkdir -p /home/ubuntu/.ssh
  cp /root/.ssh/authorized_keys /home/ubuntu/.ssh/
  chown -R ubuntu:ubuntu /home/ubuntu/.ssh
  chmod 700 /home/ubuntu/.ssh
  echo "ubuntu ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ubuntu
'
```

Variant B on realistlikum (näitab, et ei jooksuta rootina) aga mõlemad töötavad.

## 2. Playbooki käivitamine

```bash
# Pane serveri IP inventory faili
sed -i 's/YOUR_SERVER_IP/<serveri-ip>/' ansible/inventory.ini

# Kontrolli et Ansible näeb serverit
cd ansible
ansible petclinic -m ping

# Peaks vastama:
# petclinic-server | SUCCESS => { "ping": "pong" }

# Käivita playbook
ansible-playbook playbook.yml
```

Kogu playbook võtab aega umbes **8-12 minutit**, millest enamik läheb:
- apt upgrade (~2 min)
- Docker paigaldus (~1 min)
- Maven build (~5-7 min — PetClinic laeb kõik sõltuvused alla)
- Docker image'id build + käivitus (~1-2 min)

## 3. Testimine

Kui playbook on lõpetanud, kontrolli:

```bash
# Kas HTTPS töötab (-k = ignoreeri self-signed sert)
curl -k https://<serveri-ip>/ | head -20

# Peaks tagastama PetClinicu HTML-i koos "Welcome" tekstiga

# Kas HTTP suunab HTTPS-ile
curl -I http://<serveri-ip>/
# Peaks näitama: HTTP/1.1 301 Moved Permanently

# Ava brauseris
open https://<serveri-ip>/     # macOS
xdg-open https://<serveri-ip>/ # Linux
```

Brauser hoiatab sertifikaadi pärast ("Not secure") — see on ok, sest sert on self-signed. Kliki "Advanced" → "Proceed".

### Kontrolli võrgu segmenteerimine

Logi serverisse ja uuri Docker võrke:

```bash
ssh ubuntu@<serveri-ip>

# Vaata võrke
docker network ls | grep petclinic
# petclinic_frontend   bridge
# petclinic_backend    bridge

# Uuri backend võrgu omadusi — peab olema "Internal": true
docker network inspect petclinic_backend | grep -i internal
# "Internal": true,

# Vaata kes mis võrgus on
docker network inspect petclinic_frontend --format '{{range .Containers}}{{.Name}} {{end}}'
# petclinic-proxy petclinic-backend

docker network inspect petclinic_backend --format '{{range .Containers}}{{.Name}} {{end}}'
# petclinic-backend petclinic-db
```

Oodatav tulemus:
- nginx on **ainult** frontend võrgus
- db on **ainult** backend võrgus
- backend on mõlemas (bridge rollis)
- backend võrk on `internal: true`

### Proovi, et segmenteerimine päriselt tööle hakkab

```bash
# nginx proovib db-ga ühendust — peaks EBAÕNNESTUMA
docker exec petclinic-proxy nc -zv db 5432
# nc: bad address 'db' (või timeout)

# backend proovib db-ga — peaks TÖÖTAMA
docker exec petclinic-backend nc -zv db 5432 2>&1 || echo "(vajaks netcat'i installi backend image-isse)"

# db proovib internetti — peaks EBAÕNNESTUMA (internal network)
docker exec petclinic-db wget -T 3 https://google.com 2>&1 | head -3
# wget: bad address 'google.com'
```

Kui need kolm testi käituvad oodatud viisil, siis segmenteerimine on reaalselt rakendatud.

## 4. Kordus test

Käivita playbook teist korda:

```bash
ansible-playbook playbook.yml
```

Ansible kokkuvõttes peaks enamik taskidest olema **`ok`** (mitte `changed`). Normaalne on `changed`:
- `docker compose up` teatab vahel muudatusest kuigi ei teinud midagi
- apt upgrade võib leida vahepeal mõne uue paketi

Aga MITTE:
- uut kasutajat ei tohiks luua
- SSH konfi ei tohiks muuta (restart ssh handler ei tohiks käivituda)
- Maven build ei tohiks uuesti joosta (`creates:` peab selle ära hoidma)

## 5. CI/CD test

Kui teed GitHubi workflow'st eraldi haru:

```bash
git checkout -b test-ci
git commit --allow-empty -m "test deploy"
git push origin test-ci
```

Mine GitHub → Actions ja käivita `Deploy PetClinic` käsitsi (`workflow_dispatch`) test-ci haru pealt. Jälgi logi — peaks läbi jooksma samade sammude kaudu nagu lokaalselt.

Enne testi kontrolli, et secrets on paigas:
- `SSH_PRIVATE_KEY` — kogu privaatvõtme sisu
- `SSH_PUBLIC_KEY` — avalik võti
- `SERVER_HOST` — Hetzneri serveri IP
- `SERVER_USER` — `ubuntu` (või `root` kui kasutad varianti A)

## 6. Puhastus pärast testimist

Kui test läbi, kustuta Hetzner Cloudis server:
1. Console → Server → `petclinic-test`
2. Paremal üleval three-dots → **Delete**

Või `hcloud` CLI-ga:
```bash
hcloud server delete petclinic-test
```

Nii ei lähe arve suuremaks. Hetzner kustutab ka public IP, kui see pole eraldi reserveeritud.

## Probleemid millega võid kokku puutuda

**Ansible ütleb "Permission denied (publickey)"**
Kas su lokaalne `~/.ssh/id_ed25519` võti on see sama, mis Hetzneri serverisse lisati? Kontrolli `ssh-add -l` või proovi otse: `ssh -i ~/.ssh/id_ed25519 ubuntu@<ip>`.

**Maven build jookseb out-of-memory**
CX11 (2 GB RAM) pole piisav. Võta CX22 (4 GB) või suurem. Alternatiiv: lisa `06-build.yml`-i swap fail enne buildi.

**`docker compose up` ütleb "pool overlaps with other one"**
Kui oled varem sama serveriga Docker'it kasutanud, võib olla vanu võrke. Puhasta: `ssh ubuntu@<ip> 'docker network prune -f'`.

**Brauser ei näita midagi, aga curl -k töötab**
Hetzner Cloud Firewall (eraldi toode, mitte UFW) võib olla vaikimisi aktiivne ja blokeerib 443. Kontrolli Console → Firewalls.

**PetClinic vastab "502 Bad Gateway"**
Backend pole veel käima saanud (Spring Boot vajab ~30-60 sekundit start-up'iks). Oota ja vaata `docker logs petclinic-backend`.
