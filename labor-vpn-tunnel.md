# Labor: VPN tunnel (GRE) — näe, et tunnel **üksinda ei piisa**

| | Osa |
|---|-----|
| 🔧 | [Osa 1 — Topoloogia ja põhiseadistus](#osa-1--topoloogia-ja-põhiseadistus) |
| 🧠 | [Osa 2 — Mõtle enne kui seadistad](#osa-2--mõtle-enne-kui-seadistad) |
| ⚙️ | [Osa 3 — GRE tunnel](#osa-3--gre-tunnel) |
| 🔍 | [Osa 4 — Wireshark — vaata sisu](#osa-4--wireshark--vaata-sisu) |
| 💭 | [Osa 5 — Järeldused](#osa-5--järeldused) |

## Õpieesmärgid

- Ehitad topoloogia, kus **kaks sisevõrku** on eraldatud "avalikust internetist"
- Näed päriselt, miks **privaatsed IP-d ei marsruudita** üle interneti
- Seadistad **GRE tunneli** ja näed, et nüüd ping läbib
- Vaatad Wiresharkiga paketi sisu — ja näed, et **sisu on loetav**
- Sõnastad järelduse: **tunnel lahendab marsruutimise, aga mitte privaatsust**

---

## Taust

Sul on firma. Üks pool Tallinnas, teine Tartus. Mõlemas oma sisevõrk privaatsete IP-aadressidega. Vahel on **avalik internet** (ISP), mida sa ei kontrolli.

Täna teed kaks asja:

1. **Näitad, et see ei tööta:** saadad pingi PC1-st PC2-le ilma midagi tegemata → kaob kuhugi ära
2. **Lahendad pool probleemist:** seadistad GRE tunneli → ping töötab. **Aga vaatad ka, mis sees on**, ja avastad et see lahendus ei ole päriselt turvaline

Järgmisel nädalal lisame **krüpteerimise peale** (IPSec). Täna näeme, **miks** seda vaja on.

---

## Sinu aadressid

Asenda allpool `X` oma M-numbriga (M1 → X=1, M5 → X=5, ...).

| Võrk | Aadress | Selgitus |
|---|---|---|
| PC1 LAN (Tallinna pool) | 10.10.**X**.0/24 | PC1 = .10, R1 e0/1 = .1 |
| PC2 LAN (Tartu pool) | 10.20.**X**.0/24 | PC2 = .10, R3 e0/1 = .1 |
| Avalik internet — link 1 | 200.0.0.0/30 | R1 e0/0 = .1, R2 e0/0 = .2 |
| Avalik internet — link 2 | 200.0.0.4/30 | R2 e0/1 = .5, R3 e0/0 = .6 |
| GRE tunnel | 192.168.99.0/30 | R1 tunnel = .1, R3 tunnel = .2 |

> ⚠️ "Avalik internet" aadressid on **kõigil sama** — see on ühine "internet". Sisevõrkude `X` on isiklik.

---

## Osa 1 — Topoloogia ja põhiseadistus

### 1.1 Ehita topoloogia

Logi GNS3 sisse → ava projekt `lab-vpn-tunnel-<eesnimi>`.

```
[PC1] ─── e0/1 [R1] e0/0 ─── e0/0 [R2] e0/1 ─── e0/0 [R3] e0/1 ─── [PC2]
VPCS         IOL XE L3       IOL XE L3       IOL XE L3        VPCS
                              (ISP)
```

| Seade | Template | Nimi |
|---|---|---|
| R1 | IOL XE L3 | R1-SINU-NIMI |
| R2 | IOL XE L3 | R2-ISP |
| R3 | IOL XE L3 | R3-SINU-NIMI |
| PC1 | VPCS | PC1 |
| PC2 | VPCS | PC2 |

Kaablid:

| Seade A | Port | Seade B | Port |
|---|---|---|---|
| PC1 | eth0 | R1 | e0/1 |
| R1 | e0/0 | R2 | e0/0 |
| R2 | e0/1 | R3 | e0/0 |
| R3 | e0/1 | PC2 | eth0 |

Käivita kõik seadmed.

### 1.2 IP seadistus

IP seadistus pole tänase labori fookus — seadista kiiresti.

**R1:**

```
enable
configure terminal
hostname R1-SINU-NIMI
interface e0/1
 ip address 10.10.X.1 255.255.255.0
 no shutdown
 exit
interface e0/0
 ip address 200.0.0.1 255.255.255.252
 no shutdown
 exit
ip route 200.0.0.4 255.255.255.252 200.0.0.2
end
```

> Viimane rida — staatiline marsruut, et R1 teaks, kuidas R3 avaliku IP-ni jõuda.

**R2 (ISP):**

```
enable
configure terminal
hostname R2-ISP
interface e0/0
 ip address 200.0.0.2 255.255.255.252
 no shutdown
 exit
interface e0/1
 ip address 200.0.0.5 255.255.255.252
 no shutdown
 exit
end
```

> R2 ei tea **mitte midagi** sisevõrkudest. Tema on "avalik internet". Tema tunneb ainult `200.0.0.0/30` ja `200.0.0.4/30` (sest need on tema otseühendused).

**R3:**

```
enable
configure terminal
hostname R3-SINU-NIMI
interface e0/1
 ip address 10.20.X.1 255.255.255.0
 no shutdown
 exit
interface e0/0
 ip address 200.0.0.6 255.255.255.252
 no shutdown
 exit
ip route 200.0.0.0 255.255.255.252 200.0.0.5
end
```

**PC1:** `ip 10.10.X.10 255.255.255.0 10.10.X.1`

**PC2:** `ip 10.20.X.10 255.255.255.0 10.20.X.1`

### 1.3 Esimene kontroll — avalik internet

R1-lt:

```
ping 200.0.0.6
```

See peaks **töötama** — avalikud IP-d marsruutuvad läbi R2.

Kui ei tööta:
- `show ip interface brief` — kas kõik liidesed on `up/up`?
- Kas `no shutdown` puudu?
- Kas staatiline marsruut R1-l on õige?

---

## Osa 2 — Mõtle enne kui seadistad

**Ära veel midagi tee!** Vasta küsimustele (kirjuta paberile).

### 2.1 Proovi pingida sisevõrkust

PC1-l: `ping 10.20.X.10`

Mis juhtub?

___________________________________________________________________

### 2.2 Miks see nii on?

Vaata R2 marsruutimistabelit:

```
show ip route   (R2-l)
```

Kas seal on midagi `10.10.X.0/24` või `10.20.X.0/24` kohta?

___________________________________________________________________

**Miks** R2 neid võrke ei tea? Mõtle.

___________________________________________________________________

### 2.3 Lahendus?

Kas lahenduseks oleks **lisada R2-le marsruut** sisevõrkudesse? Miks see ei oleks päris elus võimalik?

___________________________________________________________________

___________________________________________________________________

> 💡 **Vihje:** R2 esindab ISP-d. Sa ei kontrolli ISP marsruutereid. Ka — privaatseid aadresseid ei marsruudita üldse avalikul internetil (RFC 1918). Sul on vaja muud lahendust.

### 2.4 Tunneli idee

Tekstist sa lugesid: **"pakk paki sisse"**. Mis see siin tähendaks?

Mis IP-aadressid oleks **välisel päisel** (mida R2 näeb)? __________________________

Mis IP-aadressid oleks **sees** (mida R2 ei vaata)? __________________________

---

## Osa 3 — GRE tunnel

Nüüd seadista GRE tunnel R1 ja R3 vahel. GRE on lihtsaim tunnelitehnoloogia — **võtab paketi ja paneb teise paketi sisse**. Krüpteerimist GRE-l ei ole (see on tahtlik — täna näeme, mida tunnel **üksinda** annab ja ei anna).

### 3.1 R1-l — loo tunnel

```
configure terminal
interface tunnel 0
 ip address 192.168.99.1 255.255.255.252
 tunnel source 200.0.0.1
 tunnel destination 200.0.0.6
 tunnel mode gre ip
 exit
ip route 10.20.X.0 255.255.255.0 192.168.99.2
end
```

**Mis siin toimub?**

- `interface tunnel 0` — virtuaalne liides, "tunneli ots"
- `tunnel source / destination` — välimine päis, **avalikud** IP-d (need on need, mida R2 näeb)
- `ip address 192.168.99.1` — tunneli sisemine "ots" — kuidas paketid tunnelisse sisenevad/väljuvad
- `ip route 10.20.X.0 ... 192.168.99.2` — "et jõuda PC2 võrku, suuna pakid tunnelisse"

### 3.2 R3-l — sama, vastupidi

```
configure terminal
interface tunnel 0
 ip address 192.168.99.2 255.255.255.252
 tunnel source 200.0.0.6
 tunnel destination 200.0.0.1
 tunnel mode gre ip
 exit
ip route 10.10.X.0 255.255.255.0 192.168.99.1
end
```

### 3.3 R2-l — **mitte midagi**

R2 ei tea endiselt sisevõrkudest mitte midagi. Tema näeb ainult avalikku liiklust kahe avaliku IP vahel (200.0.0.1 ↔ 200.0.0.6). **See ongi mõte** — ISP ei pea sisevõrkudest teadma.

### 3.4 Testi

PC1-lt:

```
ping 10.20.X.10
```

Nüüd peaks **töötama**!

Kontrolli R1-l:

```
show interfaces tunnel 0
```

Kas on `up/up`?

```
show ip route
```

Kas näed marsruuti `10.20.X.0/24` läbi `Tunnel0`?

> 🎉 **Tunneerimine töötab!** Sisevõrgu paketid läbisid avaliku interneti, kasutades GRE-d ümbrise. Marsruutimise probleem on lahendatud.
>
> **Aga oota — kas asi on päriselt turvaline?**

---

## Osa 4 — Wireshark — vaata sisu

Nüüd vaatame, **mis päriselt R1 ja R2 vahel liigub**, kui PC1 saadab PC2-le pingi.

### 4.1 Käivita capture

GNS3-s: paremklikk **R1 ↔ R2 kaablil** → **Start capture** → vali interface → Wireshark avaneb.

### 4.2 Genereeri liiklust

PC1-lt:

```
ping 10.20.X.10
```

### 4.3 Vaata pakette

Wiresharkis filtreeri: `gre` (kõik GRE paketid).

Vali üks pakett. Vaata paneeli **"Packet Details"** (keskmine paneel).

Sa peaks nägema **kolme kihti** üksteise sisse pakitud:

```
1. Väline IP päis    →   200.0.0.1  →  200.0.0.6      (mida R2 näeb)
2. GRE päis          →   "siin on midagi sees"
3. Sisemine IP päis  →   10.10.X.10 →  10.20.X.10     (originaal!)
4. ICMP andmed       →   "ping päringu sisu"
```

### 4.4 Vasta küsimustele

**Küsimus 1:** Kas sa näed sisemist IP päist (sisevõrgu aadresse)?

___________________________________________________________________

**Küsimus 2:** Kas pingi andmed (`abcdefghij...`) on loetavad?

___________________________________________________________________

**Küsimus 3:** Kujuta ette, et keegi vahel (näiteks teine ISP, teine riik) **kuulab pealt** seda liiklust. Mida ta saab teada?

___________________________________________________________________

___________________________________________________________________

**Küsimus 4:** Kas GRE tunnel **peidab** liikluse keegi vahelt? Põhjenda.

___________________________________________________________________

___________________________________________________________________

---

## Osa 5 — Järeldused

### 5.1 Mis sa täna nägid?

Tunnel lahendas üks asi: __________________________________________

Tunnel **EI** lahendanud üht asja: ___________________________________

### 5.2 Seos töölehe küsimusega 5

Töölehel sa kirjutasid:

> **"Tunneerimine ilma krüpteerimiseta lahendab .... aga ei lahenda ...."**

Kas see, mis sa täna nägid, **kinnitab** sinu vastust? Selgita.

___________________________________________________________________

___________________________________________________________________

### 5.3 Mida järgmisel nädalal lisame?

___________________________________________________________________

> 💡 Järgmisel nädalal — IPSec. See lisab tunnelile **krüpteerimise**. Wiresharkis siis vaatad uuesti, ja näed: GRE päise asemel on **ESP**, ja sisu on **jubin** — keegi vahel ei loe enam midagi.

---

## Esita tulemus

Tee screenshot või kopeeri:

1. `show ip route` R1-l (peab näitama tunneli marsruuti)
2. `show interfaces tunnel 0` R1-l (peab olema up/up)
3. PC1-l: `ping 10.20.X.10` tulemus (peab töötama)
4. **Wireshark screenshot** ühest GRE paketist (paneeli "Packet Details" lahti, et näha sisemist IP päist + ICMP andmeid)
5. Osa 4 küsimuste vastused
6. Osa 5 järelduste vastused

---

## Salvesta

R1 ja R3:

```
copy running-config startup-config
```

R2 ei muutu — salvestust pole vaja.

---

## Levinumad vead

| Viga | Sümptom | Lahendus |
|---|---|---|
| Avalik ping ei tööta | R1 ↔ R2 ping kaob | `no shutdown` puudu, või staatiline marsruut puudu |
| Tunnel jääb `down/down` | `show int tun0` | `tunnel source / destination` IP-d valed, kontrolli avalikud aadressid |
| Ping läbi tunneli ei tööta | Tunnel up, aga ping ei lähe | `ip route 10.X.0 ... 192.168.99.X` puudu — kõikide võrkudeni marsruut peab tunnelist läbi minema |
| Wireshark näitab ICMP, mitte GRE | Vale link valitud | Capture **R1↔R2** linkilt, mitte PC↔R1 linkilt |
| MTU probleemid (suured paketid katki) | Ping töötab, http ei | `ip mtu 1400` tunneli liidesele (kursusel pole vaja, aga päris elus jah) |
