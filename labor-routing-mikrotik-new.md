# Labor: Staatiline routing MikroTikus (edasijõudnutele)

| | Osa |
|---|-----|
| 🔧 | [Osa 1 — Topoloogia ja algkonfiguratsioon](#osa-1--topoloogia-ja-algkonfiguratsioon) |
| 🧠 | [Osa 2 — Mõtle enne kui seadistad](#osa-2--mõtle-enne-kui-seadistad) |
| ⚙️ | [Osa 3 — Lisa staatilised marsruudid](#osa-3--lisa-staatilised-marsruudid) |
| 📊 | [Osa 4 — Analüüsi marsruutimistabelit](#osa-4--analüüsi-marsruutimistabelit) |
| ⚙️ | [Osa 5 — Default route](#osa-5--default-route) |
| 🔴 | [Osa 6 — Riku ja paranda](#osa-6--riku-ja-paranda) |
| 🆚 | [Bonus — Cisco vs MikroTik võrdlus](#bonus--cisco-vs-mikrotik-võrdlus) |
| 🖱️ | [Boonus 2 — sama asi WinBox-iga](#boonus--sama-asi-winbox-iga-kui-jõuad) |

## Õpieesmärgid

- Ehitad sama routing topoloogia mida tegid Cisco IOL-iga, **aga MikroTik RouterOS-iga**
- Mõistad et **routing on mõiste, mitte käsk** — sama loogika, erinev süntaks
- Loed MikroTik dokumentatsiooni iseseisvalt
- Võrdled kahte platvormi ja näed kuhu kontseptsioon liigub

---

## Taust

Cisco on tugev kontserni- ja datacentri-turul, aga **Eesti SMB sektoris** on MikroTik palju levinum. Pea iga väikefirma võrk siin jookseb MikroTikuga. Tööturul väga tõenäoline et kohtad seda.

MikroTik kasutab oma operatsioonisüsteemi RouterOS — süntaks on **täiesti erinev Ciscost**. See on tahtlik valik:
- Sa juba tead **mis on staatiline marsruut** (Cisco labor)
- Nüüd saad aru et mõiste eksisteerib **käsust sõltumatult**
- Kui sa loed dokumentatsiooni ja saad MikroTikus sama topoloogia tööle — siis sa **päriselt** mõistad routingut

---

## Sinu aadressid

| | LAN-A (PC1) | LAN-C (PC3) |
|---|---|---|
| M1 | 10.10.**1**.0/24 | 30.30.**1**.0/24 |
| M2 | 10.10.**2**.0/24 | 30.30.**2**.0/24 |
| M3 | 10.10.**3**.0/24 | 30.30.**3**.0/24 |
| ... | ... | ... |
| M17 | 10.10.**17**.0/24 | 30.30.**17**.0/24 |

Transit võrgud kõigil samad: `172.16.1.0/30` (R1↔R2) ja `172.16.2.0/30` (R2↔R3)

> ⚠️ Asenda allpool `X` oma M-numbriga.

---

## Osa 1 — Topoloogia ja algkonfiguratsioon

### 1.1 Ehita topoloogia

```
                   172.16.1.0/30        172.16.2.0/30
[PC1]──[SW1]──e1[R1]e2 ──────── e1[R2]e2 ──────── e1[R3]e2──[SW3]──[PC3]
        LAN-A   .1   .1              .2  .1              .2   .1   LAN-C
   10.10.X.0/24                                              30.30.X.0/24
```

| Seade | Template | Nimi |
|-------|----------|------|
| R1 | mikrotik-chr | R1-SINU-NIMI |
| R2 | mikrotik-chr | R2-SINU-NIMI |
| R3 | mikrotik-chr | R3-SINU-NIMI |
| SW1 | IOL XE L2 | SW1-SINU-NIMI |
| SW3 | IOL XE L2 | SW3-SINU-NIMI |
| PC1 | VPCS | PC1 |
| PC3 | VPCS | PC3 |

Kaablid:

| Seade A | Port | Seade B | Port |
|---------|------|---------|------|
| SW1 | e0/0 | R1 | ether1 |
| SW1 | e0/1 | PC1 | eth0 |
| R1 | ether2 | R2 | ether1 |
| R2 | ether2 | R3 | ether1 |
| R3 | ether2 | SW3 | e0/0 |
| SW3 | e0/1 | PC3 | eth0 |

Käivita kõik seadmed ja oota ~60 sekundit.

### 1.2 MikroTik sisselogimine

Ava R1 konsool (paremkliki → **Web console**).

Esimene login:
- Username: `admin`
- Password: (tühi — vajuta Enter)

CHR küsib uut parooli — pane `lab2025` (lihtne, see on labor). Sama tee R2 ja R3-ga.

> 💡 RouterOS prompt näeb välja `[admin@MikroTik] >`. Kõik käsud algavad `/` märgiga (käsupuu juur). Tab-klahv täiendab käske ja `?` näitab abi.

### 1.3 IP seadistus

MikroTik süntaks on uus, seetõttu IP-käsud on **antud**. Routing tuleb hiljem **iseseisvalt**.

**R1:**

```
/system identity set name=R1-SINU-NIMI
/ip address add address=10.10.X.1/24 interface=ether1
/ip address add address=172.16.1.1/30 interface=ether2
/ip address print
```

**R2:**

```
/system identity set name=R2-SINU-NIMI
/ip address add address=172.16.1.2/30 interface=ether1
/ip address add address=172.16.2.1/30 interface=ether2
/ip address print
```

**R3:**

```
/system identity set name=R3-SINU-NIMI
/ip address add address=172.16.2.2/30 interface=ether1
/ip address add address=30.30.X.1/24 interface=ether2
/ip address print
```

**SW1** ja **SW3** — pane hostname, muud pole vaja (tavaline L2 switch).

**PC1:** `ip 10.10.X.10 255.255.255.0 10.10.X.1`

**PC3:** `ip 30.30.X.10 255.255.255.0 30.30.X.1`

### 1.4 Kontrolli põhiühendust

PC1-l: `ping 10.10.X.1` — peaks töötama (oma gateway).

R1-l: `/ping 172.16.1.2` — peaks töötama (otsene naaber R2).

R2-l: `/ping 172.16.2.2` — peaks töötama (otsene naaber R3).

R3-l: `/ping 30.30.X.10` — peaks töötama (oma LAN).

**Aga:** PC1-l `ping 30.30.X.10` — **ei tööta**. See ongi labori algus. Miks ei tööta?

---

## Osa 2 — Mõtle enne kui seadistad

**Ära veel midagi seadista!** Vasta kõigepealt nendele küsimustele paberile:

1. Iga ruuter teab praegu **ainult oma otseühenduses olevaid võrke**. Vaata `/ip route print` igal kolmel ruuteril. Mis võrgud seal on? Mis staatus neil on?

2. PC1 saadab paketi sihtkohale `30.30.X.10` → R1 saab paketi. Mida R1 teeb kui ta vaatab oma marsruutimistabelit? (vihje: ta ei leia sellele võrgule rida — mis edasi?)

3. Iga ruuter peab teadma **igast võrgust mida ta otse ei näe**. Loe topoloogia üle. Kui mitu lisa-marsruuti peab näiteks **R2 juurde õppima**?

4. Sa ei kirjuta kunagi marsruuti otse võrgu poole — vaid **kuhu pakett edasi saata** (next-hop). Mis on **R1 jaoks next-hop** kui ta tahab jõuda `30.30.X.0/24` võrku?

5. Kui kõik kolm ruuterit saavad oma marsruudid kätte, siis PC1 ping läheb läbi. Aga mis juhtub vastusega? Mis peavad **R3 ja R2** teadma, et vastus jõuaks tagasi PC1-ni?

<details>
<summary>❓ Jäid hätta? Kliki vihjele.</summary>

Loe loengumaterjalina `routing_kordamine.md` (oppematerjalid repos). Iga küsimus on seal mainitud. Eriti küsimus 5 — see on **kõige sagedasem viga** alustaja puhul.

</details>

---

## Osa 3 — Lisa staatilised marsruudid

Nüüd seadista routing — **iseseisvalt**. Kokku **kuus marsruuti**:

| Ruuter | Mis võrgule | Next-hop |
|--------|-------------|----------|
| R1 | 172.16.2.0/30 | ? |
| R1 | 30.30.X.0/24 | ? |
| R2 | 10.10.X.0/24 | ? |
| R2 | 30.30.X.0/24 | ? |
| R3 | 172.16.1.0/30 | ? |
| R3 | 10.10.X.0/24 | ? |

Next-hop on alati **sinu otsese naabri IP**. Mõtle vaata kummal pool naaber asub.

**Käske ei ole.** Loe MikroTik docs:

📖 https://help.mikrotik.com/docs/spaces/ROS/pages/328088/IP+Routing

Otsi sealt **"Static Routes"** ja **"How to add a route"**.

Või konsoolis kasuta `?`:

```
/ip route add ?
```

See näitab kõik parameetrid. Sama trikk töötas Ciscos ka (`?` käsu lõpus).

<details>
<summary>❓ Ei leia õiget süntaksit? Kliki vihjele.</summary>

Otsi MikroTik docs-ist võtmesõnu **`dst-address`** ja **`gateway`**. Ülejäänu on Tab-klahviga loetav.

</details>

Pärast iga marsruudi lisamist verifitseeri:

```
/ip route print
```

### Test

PC1-lt:

```
ping 30.30.X.10
trace 30.30.X.10
```

`trace` peaks näitama paketi teekonda läbi kõigi 3 ruuteri.

<details>
<summary>❓ Ping ei läbi? Proovi ise lahendada, siis kliki.</summary>

- Kas kõik 6 marsruuti on lisatud? Iga ruuter, vaata `/ip route print` — peaks olema mõlema kaugema võrgu rida.
- Kas next-hop'id on õiged? Pane tähele: R1 ja R3 jaoks on next-hop **R2 IP** (mitte R3 ega R1 enda IP).
- Kas pakett läheb sihtkohani aga **vastus ei tule tagasi**? See on klassikaline viga — mõlemas suunas peavad marsruudid olema. (Vaata Osa 2 küsimus 5.)

</details>

---

## Osa 4 — Analüüsi marsruutimistabelit

R2-l (transit ruuter):

```
/ip route print detail
```

### Täida tabel

Vali tabelist 4-5 kõige huvitavamat rida ja kirjuta välja:

| DST-ADDRESS | GATEWAY | DISTANCE | FLAGS (DAc, As, jne) |
|-------------|---------|----------|----------------------|
| | | | |

### Vasta küsimustele

1. Mis tähendab `DAc` flag? Lahti seletatuna: `D` = dynamic, `A` = active, `c` = connected. Kuidas see erineb `As`-st?

2. Kas connected marsruudid on **alati** routing tables, isegi kui sa neid ei lisanud? Miks?

3. Mis on `DISTANCE` veerg? Kuidas see seostub Cisco AD (Administrative Distance) mõistega?

4. Kui sa lisaksid sama sihtkoha jaoks **kaks erineva distance'iga marsruuti**, kumb läheks aktiivseks (`A`-flagiga)? Proovi katsetada — lisa R1-le 30.30.X.0/24 marsruut distance=200 läbi mingi vale gateway. Vaata mis juhtub `/ip route print` väljundis. Eemalda pärast.

---

## Osa 5 — Default route

Kustuta R1-st marsruut 30.30.X.0/24:

```
/ip route remove [find dst-address=30.30.X.0/24]
```

Lisa selle asemele **default route** — marsruut, mis kehtib kõigele mis pole spetsiifiliselt mainitud:

```
/ip route add dst-address=0.0.0.0/0 gateway=172.16.1.2
```

PC1-lt: `ping 30.30.X.10` — kas töötab?

### Vasta küsimustele

1. Miks default route töötas, kuigi `30.30.X.0/24` pole tabelisse spetsiifiliselt lisatud?

2. Kus reaalses elus default route'i kasutatakse? (vihje: sinu kodu ruuteri tabelis on praegu üks selline)

3. Mis juhtub kui mõlemad — spetsiifiline ja default — on samaaegselt tabelis? Lisa R1-le ka `30.30.X.0/24` tagasi (näiteks distance=10) ja vaata `/ip route print`. Kumb on aktiivne ja miks?

---

## Osa 6 — Riku ja paranda

### Harjutus A — Vale next-hop

R2-l muuda 10.10.X.0/24 marsruudi gateway vääraks. Esmalt vaata olemasoleva marsruudi indeksit:

```
/ip route print
```

Siis muuda:

```
/ip route set [find dst-address=10.10.X.0/24] gateway=192.0.2.99
```

PC1-lt: `ping 30.30.X.10`. Mis juhtub? Üks vale ruuter rikub kõike?

Paranda tagasi (õige gateway = R1 IP transit-võrgus).

### Harjutus B — Eemalda tagasitee

R3-st **eemalda** marsruut 10.10.X.0/24-le:

```
/ip route remove [find dst-address=10.10.X.0/24]
```

PC1-lt: `ping 30.30.X.10`. 

- Kas pakett **lähebki** sihtkohta (PC3-ni)? Kontrolli `trace` käsuga.
- Kas **vastus tuleb tagasi**?

Mis see ütleb routingust? See on Osa 2 küsimuse 5 vastus — pakett peab teadma teed **mõlemas suunas**.

Paranda tagasi.

---

## Bonus — Cisco vs MikroTik võrdlus

Täida tabel oma kogemuse põhjal (osad on antud, osad pead ise meenutama/leidma):

| Tegevus | Cisco IOS | MikroTik RouterOS |
|---------|-----------|-------------------|
| Hostname | `hostname R1` | `/system identity set name=R1` |
| IP liidesele | `interface e0/0`<br>`ip address 10.0.0.1 255.255.255.0`<br>`no shutdown` | ? |
| Staatiline marsruut | `ip route 10.0.0.0 255.255.255.0 192.168.1.1` | ? |
| Default route | `ip route 0.0.0.0 0.0.0.0 192.168.1.1` | ? |
| Vaata routing table | `show ip route` | ? |
| Kustuta marsruut | `no ip route 10.0.0.0 255.255.255.0 192.168.1.1` | `/ip route remove [find dst-address=...]` |
| Salvesta config | `copy running-config startup-config` | (MikroTikus salvestub automaatselt) |

### Lõppmõte

Mõlemas süsteemis tegid sa **sama asja**:

1. Otseühenduses võrgud lisanduvad automaatselt (connected)
2. Kaugemate võrkude jaoks lisad next-hop'i (static)
3. Üldine "muu kõik" reegel — default route

Süntaks erines. Kontseptsioon ei. **See on routing.**

---

## Esita tulemus

Tee screenshot või kopeeri tekst:

1. `/ip route print` igal kolmel ruuteril (pärast Osa 3)
2. `/ip route print detail` R2-l (Osa 4)
3. PC1-lt `trace 30.30.X.10` täielik output
4. Cisco vs MikroTik tabel täidetuna
5. Osa 2 küsimuste vastused

---

## Salvesta

MikroTikus salvestub konfiguratsioon **automaatselt** — pole eraldi käsku vaja. Cisco IOL-i `copy running-config startup-config` siin pole.

---

## Boonus — sama asi WinBox-iga (kui jõuad)

Eestis kasutavad pea kõik MikroTiku haldajad **WinBoxi** — graafilist tööriista. CLI on tugev õpetamise jaoks (näed mõisteid), aga päris elus tihti klikitakse pilti. Vaatame nüüd **kuidas sama konfiguratsioon näeb GUI-st**.

### Eelduseks

Sinu kolme CHR ruuteri konfiguratsioon on töökorras (sa just lõpetasid Osa 3-6). Ära midagi maha võta.

### 1. Lisa WinBox topoloogiasse

GNS3 vasakus paneelis **Browse all devices** → otsi **MikroTik WinBox 2026**. Lohista töölauale.

WinBox node on Docker container — käivitub kiiresti (~10 sek).

### 2. Ühenda WinBox sisevõrku

WinBox vajab IP-d kliendi võrgus, et CHR-i näha. Ühenda kaabliga:

```
WinBox ── SW1 (e0/3) ── (juba olemasolev klient-võrk LAN-A)
```

Kui WinBox node käivitub esimesena, ta saab automaatselt IP DHCP kaudu — aga kui sinu topoloogias DHCP-d pole (mu labor seda ei kasuta), pead käsitsi seadma.

WinBox node Web console:
```
ip addr add 10.10.X.50/24 dev eth0
ip route add default via 10.10.X.1
```

### 3. Käivita WinBox GUI

Paremkliki WinBox node → **Web console** → avaneb brauseriaknas Linux desktop noVNC kaudu.

Topeltkliki **WinBox** ikoonil. Avaneb ühendusedialoog.

### 4. Ühenda R1-ga

Ühendusakna ülaosas:

| Väli | Väärtus |
|------|---------|
| Connect to | `10.10.X.1` (R1 LAN-A liides) |
| Login | `admin` |
| Password | `lab2025` |

Klik **Connect**.

> 💡 **Vihje:** WinBox oskab ka MAC address discovery — kliki "Neighbors" tab, näed kõiki MikroTikuid samas L2-segmendis. Mugav esmaseks seadistamiseks kui IP-d veel pole.

### 5. Vaata routing tabelit GUI-st

Vasakul menüüs: **IP → Routes**

Sa peaksid nägema **täpselt sama nimekirja** mida nägid CLI-s `/ip route print` käsuga. Lihtsalt nüüd lahter-vormis.

| Mida näed CLI-s | Mida näed WinBoxis |
|-----------------|---------------------|
| `DAc` flag | "Active" + "Connect" linnukesed |
| `As` flag | "Active" + "Static" linnukesed |
| `dst-address` veerg | "Dst. Address" lahter |
| `gateway` veerg | "Gateway" lahter |
| `distance` veerg | "Distance" lahter |

### 6. Lisa marsruut WinBox-ist

Kliki **+** (uus marsruut) ülemises tööribas. Avaneb dialoog. Tee proov:
- Dst. Address: `192.0.2.0/24`
- Gateway: `172.16.1.2`
- (jätta muu vaikeväärtustele)
- **OK**

Vaata IP → Routes — uus rida ilmus.

Nüüd kustuta see WinBox-ist (paremkliki rida → Remove) ja vaata CLI-s `/ip route print` — kas sealt on ka kadunud?

### 7. Mõte

| | CLI | WinBox |
|---|-----|--------|
| Õpetab mõisteid | ✅ | ❌ (klikkimine ei seleta miks) |
| Kiirem rutiintööks | ❌ | ✅ |
| Töötab SSH-ga remote | ✅ | ❌ (vajab GUI ühendust) |
| Tugevam skriptimiseks | ✅ | ❌ |
| Päris-elu Eesti SMB | osaliselt | ✅ |
| Vea kohe näha | ✅ | ✅ |

Kummalgi on koht. **Mõista kontseptsiooni CLI kaudu — kasuta WinBoxi rutiinis.** Kui sa CLI-d ei oska, siis WinBox-i klikid muudavad sind kasvõi targemaks ainult juhuse korral. Kui sa mõistad — WinBox lihtsalt säästab aega.

### 8. Esita boonusena

Tee screenshot WinBox **IP → Routes** aknast (R1 või R2 või R3 — sinu valik). Lisa esituse juurde.
