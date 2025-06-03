# LinEmpire Sistēmas Administratora Uzdevums

Šis uzdevums simulē sistēmas administratora lomu nelielā IT uzņēmumā **"LinEmpire"**, kurā jākonfigurē Linux sistēma, lai nodrošinātu drošu un efektīvu lietotāju un tiesību pārvaldību dažādām komandām.

---

## Projekta pārskats

Uzņēmumā "LinEmpire" strādā dažādas komandas ar atšķirīgām piekļuves vajadzībām:

- **Izstrādātāji (Dev):** piekļuve koda direktorijai, bet ne produkcijas konfigurācijas failiem.
- **Testētāji (QA):** lasīšana lietojumprogrammu žurnāliem, bez iespējas rediģēt kodu.
- **Operacionālie inženieri (Ops):** plaša piekļuve, ieskaitot pakalpojumu restartēšanu.
- **Servisu konti:** atsevišķi konti dažādiem pakalpojumiem (piemēram, tīmekļa serverim, datu bāzei).

---

## Uzdevums

Konfigurēt Linux sistēmu tā, lai katra nodaļa efektīvi veiktu savus pienākumus, novēršot kļūdas un nesankcionētas darbības.

---

## Darbību plāns

### 1. Vides sagatavošana

- Izveidot VM (Ubuntu/Debian/CentOS) vai mākoņa instanci.

### 2. Lietotāju un grupu izveide

- Grupas: `devteam`, `qateam`, `ops`
- Lietotāji:
  - `alice`, `alex` → `devteam`
  - `bob` → `qateam`
  - `charlie` → `ops`
- Nodrošināt `/etc/skel` failu kopēšanu.
- Paroļu politika: maiņa pēc noteikta laika.

### 3. Grupas un tiesības

- `/srv/project/`:
  - lasīšana/rakstīšana `devteam`
  - ierobežota piekļuve citiem
- `/srv/logs/`:
  - lasīšana `qateam`, `devteam`
  - pilna piekļuve `ops`
- SGID bits uz `/srv/project/`
- Sticky bit uz `/tmp/sharedtest/`

### 4. Paplašinātie ACL saraksti

- `alex`: pilna piekļuve `/srv/logs/app1/`
- citi `devteam`: tikai lasīšana
- `setfacl`, `getfacl` + pareiza `mask`

### 5. Pakalpojumu konti un drošība

- Lietotājs `nginx` (UID < 1000, bez shell)
- Pieeja `/var/www/`

### 6. Root un sudo

- Aizliegt `root` SSH pieteikšanos.
- `sudoers` konfigurācija:
  - `charlie`: restartēt `nginx`
  - `bob`: lasīt `/var/log/auth.log`
  - `alice`: `sudo` visas komandas ar paroles ievadi

### 7. `umask` un failu izveide

- Iestatīt `umask` uz `027` vai `002`
- Nodrošināt atbilstošas failu tiesības

### 8. Masveida reģistrācijas automatizācija (bonuss)

- Skripts (Bash/Python):
  - pievieno lietotājus `devteam`
  - ģenerē paroles vai SSH atslēgas
  - paroles maiņa pie pirmā pieteikšanās

### 9. Žurnāli un audits

- Pārbaudīt `/var/log/auth.log` vai `/var/log/secure`
- Izmantot `last` komandu
- (Papildu) konfigurēt `auditd` žurnālam par `/srv/logs/`

## Risinājums

Zemāk pieejams skripts, kas izpilda uzdevumu Ubuntu/Debian vidē.

### Bash skripts

```bash
#!/bin/bash
# LinEmpire sistēmas konfigurācijas skripts

# --- 1. Vides sagatavošana ---
sudo apt update -y
sudo apt install -y acl nginx auditd

# --- 2. Grupas un lietotāji ---
groupadd devteam
groupadd qateam
groupadd ops

useradd -m -g devteam alice
passwd alice
chage -d 0 alice

useradd -m -g devteam alex
passwd alex
chage -d 0 alex

useradd -m -g qateam bob
passwd bob
chage -d 0 bob

useradd -m -g ops charlie
passwd charlie
chage -d 0 charlie

# --- 3. Tiesību konfigurācija ---
mkdir -p /srv/project/
chown :devteam /srv/project/
chmod 2770 /srv/project/

mkdir -p /srv/logs/
chown :ops /srv/logs/
chmod 2770 /srv/logs/
setfacl -m d:g:devteam:r-x /srv/logs/
setfacl -m d:g:qateam:r-x /srv/logs/

mkdir -p /tmp/sharedtest/
chmod 1777 /tmp/sharedtest/

# --- 4. ACL Alex žurnāliem ---
mkdir -p /srv/logs/app1/
chown :ops /srv/logs/app1/
chmod 2770 /srv/logs/app1/
setfacl -m g:devteam:r-x /srv/logs/app1/
setfacl -m d:u:alex:rwx /srv/logs/app1/

# --- 5. Pakalpojumu konts ---
useradd -r -s /usr/sbin/nologin nginx
mkdir -p /var/www/html/
chown -R nginx:nginx /var/www/html/
chmod -R 755 /var/www/html/
echo "<h1>Welcome to LinEmpire!</h1>" > /var/www/html/index.html
chown nginx:nginx /var/www/html/index.html
chmod 644 /var/www/html/index.html

# --- 6. sudoers konfigurācija ---
echo 'charlie ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx' > /etc/sudoers.d/charlie_ops
chmod 0440 /etc/sudoers.d/charlie_ops

echo 'bob ALL=(ALL) NOPASSWD: /usr/bin/cat /var/log/auth.log, /usr/bin/less /var/log/auth.log, /usr/bin/tail -f /var/log/auth.log' > /etc/sudoers.d/bob_qa
chmod 0440 /etc/sudoers.d/bob_qa

echo 'alice ALL=(ALL) ALL' > /etc/sudoers.d/alice_dev
chmod 0440 /etc/sudoers.d/alice_dev

# --- 7. umask konfigurācija ---
echo "UMASK 002" >> /etc/login.defs
echo "umask 002" > /etc/profile.d/custom_umask.sh
chmod +x /etc/profile.d/custom_umask.sh

# --- 8. Automātiskā lietotāju reģistrācija ---
cat << 'EOF' > /usr/local/bin/onboard_new_devs.sh
#!/bin/bash
GROUP="devteam"
PASSWORD_LENGTH=12
for USERNAME in "$@"; do
    useradd -m -g "$GROUP" "$USERNAME"
    PASS=$(openssl rand -base64 $PASSWORD_LENGTH | head -c $PASSWORD_LENGTH)
    echo "$USERNAME:$PASS" | chpasswd
    chage -d 0 "$USERNAME"
    echo "$USERNAME parole: $PASS"
done
EOF

chmod +x /usr/local/bin/onboard_new_devs.sh

# --- 9. Audits ---
auditctl -w /srv/logs/ -p rwxa -k project_logs
```
