# GNS3 Demo — Õpetaja juhend (5-7 min ette näitamine)

## Enne tundi (ettevalmistus)

1. Logi sisse `admin` / `Admin2025!` aadressil `http://192.168.5.110:3080`
2. Loo uus projekt: **demo-nat**
3. Projekeeri ekraan tudengitele
4. Joonista tahvlile (või näita ekraanil) tänane topoloogia:

```
    [PC1] ──── e0/1 [SW1] e0/0 ──── e0/0 [R1] e0/1 ──── e0/0 [SERVER]
    VPCS              IOL XE L2           IOL XE L3           VPCS
   10.10.10.10                                            200.0.0.100
                                 10.10.10.1  200.0.0.1
                                  (inside)    (outside)
```

> "Tänane eesmärk — teie ehitate selle võrgu üles ja seadistate NAT/PAT. Lõpuks peavad PC1 ja PC2 pääsema serverisse, kasutades ühte avalikku IP-d."

---

## Demo skript

### 1. Sisselogimine (30 sek)

> "Avage brauser, minge aadressile `192.168.5.110:3080`. Kasutajanimi on teie eesnimi väiketähtedega, parool `av2lab2025`."

Näita ette: logi sisse, kliki oma projektil.

---

### 2. Seadme lisamine (1 min)

> "Vasakul üleval on pluss-ikoon. Kliki sellel."

Kliki **+** ikoonil → avaneb seadmete nimekiri.

> "Siin on meie seadmed. Lohista töölauale."

**Lohista järjest:**

1. **IOL XE L3** → "See on ruuter, R1. Siin seadistame NAT-i."
2. **IOL XE L2** → "Switch, lihtsalt ühendab arvutid ruuteriga."
3. **VPCS** → "Lihtne arvuti pingimiseks." Lohista kaks tükki.

Tulemus ekraanil: 4 seadet töölaual.

---

### 3. Ümbernimetamine (30 sek)

> "Topeltkliki seadme NIMEL — mitte ikoonil, vaid tekstil."

Kliki `IOU1` nimel → muuda → `R1-MARIA`

> "Teie nimi peab olema sees — see tõestab et teie tegite töö, mitte naaber."

---

### 4. Kaablite ühendamine (1 min)

> "Vasakul on kaabli ikoon."

Kliki kaabli ikoonil (joon kahe punktiga).

> "Nüüd kliki esimesel seadmel..."

Kliki **R1-MARIA** → vali **Ethernet0/0**

> "...ja siis teisel seadmel."

Kliki **SW1** → vali **Ethernet0/0**

Kaabel tekib!

> "Kui teete vale ühenduse — paremkliki kablil, Delete."

Vajuta **ESC** et lõpetada kaablite režiim.

Ühenda ka: SW1 `e0/1` → PC1 `eth0`

---

### 5. Käivitamine (30 sek)

> "Roheline kolmnurk üleval — Start. Käivitab kõik seadmed korraga."

Vajuta **▶ Start**.

> "Oodake umbes minut. Kui näete rohelist täppi — seade töötab."

Oota kuni R1-l on roheline täpp.

---

### 6. Konsooli avamine ja käsud (2 min)

> "Paremkliki seadmel → Web console."

Paremkliki **R1-MARIA** → **Web console**

Uus aken avaneb. Kui ekraan on tühi:

> "Kui on must ekraan — vajutage Enter."

Vajuta Enter. Peaks nägema:

```
Router>
```

> "Nüüd saame ruuterit seadistada. Vaatame liideste olekut:"

Kirjuta:

```
enable
show ip interface brief
```

Näita tulemust:

```
Interface        IP-Address      OK? Method Status                Protocol
Ethernet0/0      unassigned      YES unset  up                    up
Ethernet0/1      unassigned      YES unset  up                    up
```

> "Näete — liidesed on üleval, aga IP aadressid on seadistamata. See ongi teie ülesanne."

Kirjuta kiirelt IP ühele liidesele:

```
configure terminal
interface e0/0
 ip address 10.10.10.1 255.255.255.0
 exit
end
show ip interface brief
```

> "Nüüd on `e0/0`-l aadress olemas. Ülejäänu seadistate juhendi järgi."

---

### 7. Peatamine (15 sek)

> "Punane ruut üleval — Stop. Peatab kõik seadmed. Projekt jääb alles."

**Ära vajuta** — jäta demo projekt käima kuni tudengid alustavad.

---

## Pärast demot — juhised tudengitele

Ütle:

> "Nüüd teie kord. Teil on kaks dokumenti:"
>
> "Esimene — **tudengi juhend**. Seal on kirjas kuidas GNS3-d kasutada: sisselogimine, seadmed, kaablid, konsool."
>
> "Teine — **NAT labor**. See on tänane ülesanne. Viis osa."
>
> "Osa 1 — ehitate topoloogia. IP käsud on antud."
>
> "Osa 2 — mõtlemisülesanne. Vastake küsimustele ENNE kui hakkate NATi seadistama."
>
> "Osa 3 — seadistate PAT **iseseisvalt**. Käske ei ole. Kasutage loengumaterjalina `02_nat_seadistamine.md` — sealt leiate kõik vajaliku."
>
> "Kui jääte hätta — vaadake kõigepealt materjali. Kui ikka ei saa — tõstke käsi."

Jaga materjalid (link / printida / ekraan).

---

## Varuplaanid

| Probleem | Mida teha |
|----------|-----------|
| GNS3 ei lae / error | `ssh almalinux@192.168.100.110 "sudo systemctl restart gns3"`, oota 30 sek |
| Tudeng ei saa sisse | Kasutaja: eesnimi väiketähtedega. Parool: `av2lab2025` |
| Konsool ei avane (popup blocked) | Chrome → luba popupid `192.168.5.110` jaoks |
| IOL ei booti (punane täpp) | Oota 60 sek. Kui ikka → paremkliki → Stop, oota 5 sek, Start |
| "Port is already used" | Vana kaabel on pordis. Paremkliki kablil → Delete |
| Server aeglane / RAM täis | `ssh almalinux@192.168.100.110 "free -h"` |
| Tudeng lõhkus midagi | Admin kontoga saab vaadata kõiki projekte |
| VPN ühendus katkes (kaugtöö) | Ühenda VPN uuesti, `192.168.100.110:3080` |

---

## Pärast tundi

1. Peata demo projekt: ava `demo-nat` → Stop all → kustuta projekt
2. Kontrolli serveri olekut: `ssh almalinux@192.168.100.110 "free -h && ps aux | grep -c iou"`