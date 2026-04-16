# Labor: DHCP ja DNS seadistamine (dnsmasq)

| | Osa |
|---|-----|
| 🔧 | [Osa 1 — Topoloogia ja põhiseadistus](#osa-1--topoloogia-ja-põhiseadistus) |
| 🧠 | [Osa 2 — Mõtle enne kui seadistad](#osa-2--mõtle-enne-kui-seadistad) |
| ⚙️ | [Osa 3 — Seadista DHCP](#osa-3--seadista-dhcp) |
| ⚙️ | [Osa 4 — Seadista DNS](#osa-4--seadista-dns) |
| 📊 | [Osa 5 — Analüüsi tulemusi](#osa-5--analüüsi-tulemusi) |
| 🔴 | [Osa 6 — Riku ja paranda](#osa-6--riku-ja-paranda) |

## Õpieesmärgid

- Ehitad võrgu, kus klientarvutid saavad IP-aadressi **automaatselt** DHCP kaudu
- Seadistad dnsmasq serveri DHCP ja DNS teenuseks **iseseisvalt**
- Konfigureerid ruuteril DHCP relay (`ip helper-address`)
- Testid DNS nimeresolutsiooni
- Leiad ja parandad DHCP/DNS seadistuse vigu

---

## Taust

Sul on firma, kus on 2 arvutit kliendivõrgus. Eraldi serverivõrgus (`172.16.0.0/24`) jookseb dnsmasq — DHCP ja DNS server. Ruuter ühendab kaks võrku. Sinu ülesanne: seadista kõik nii, et arvutid saavad IP-aadressi automaatselt ja saavad nimepäringuid teha.

### Sinu aadressid

Igal tudengil on oma kliendivõrk vastavalt loosimissedeli numbrile. **Serverivõrk on kõigil sama.**

| Sinu M-number | Kliendivõrk | R1 e0/0 (gateway) | DHCP vahemik |
|---------------|-------------|---------------------|--------------|
| M1 | 10.10.**1**.0/24 | 10.10.**1**.1 | 10.10.**1**.100 – .200 |
| M2 | 10.10.**2**.0/24 | 10.10.**2**.1 | 10.10.**2**.100 – .200 |
| M3 | 10.10.**3**.0/24 | 10.10.**3**.1 | 10.10.**3**.100 – .200 |
| ... | ... | ... | ... |
| M17 | 10.10.**17**.0/24 | 10.10.**17**.1 | 10.10.**17**.100 – .200 |

Serverivõrk kõigil: `172.16.0.0/24` — R1 e0/1 = `172.16.0.1`, DNSMASQ = `172.16.0.10`

> ⚠️ Asenda allpool `X` oma sedeli numbriga. Nt kui sa oled **M5**, siis sinu kliendivõrk on `10.10.5.0/24`.

---

## Osa 1 — Topoloogia ja põhiseadistus

### 1.1 Ehita topoloogia

Logi GNS3 sisse ja ava oma projekt. Ehita järgmine võrk:

```
    [PC1] ──── e0/1 [SW1] e0/0 ──── e0/0 [R1] e0/1 ──── eth0 [DNSMASQ]
    VPCS              IOL XE L2           IOL XE L3         Docker
    [PC2] ──── e0/2            10.10.X.1    172.16.0.1   172.16.0.10
```

| Seade | Template | Nimi |
|-------|----------|------|
| R1 | IOL XE L3 | R1-SINU-NIMI |
| SW1 | IOL XE L2 | SW1-SINU-NIMI |
| PC1 | VPCS | PC1 |
| PC2 | VPCS | PC2 |
| DNSMASQ | dnsmasq | DNSMASQ |

Kaablid:

| Seade A | Port | Seade B | Port |
|---------|------|---------|------|
| SW1 | e0/0 | R1 | e0/0 |
| SW1 | e0/1 | PC1 | eth0 |
| SW1 | e0/2 | PC2 | eth0 |
| R1 | e0/1 | DNSMASQ | eth0 |

Käivita kõik seadmed ja oota ~60 sekundit.

### 1.2 IP seadistus

IP seadistus pole selle labori teema — fookus on DHCP-l ja DNS-il. Aga sa pead võrgu üles panema enne kui DHCP-d seadistad.

**R1** — seadista hostname ja kaks liidest:

| Liides | IP aadress | Roll |
|--------|-----------|------|
| e0/0 | 10.10.X.1 /24 | Kliendivõrgu gateway |
| e0/1 | 172.16.0.1 /24 | Serverivõrgu gateway |

<details>
<summary>❓ Ei mäleta kuidas IOS-is liidest seadistada? Kliki.</summary>

```
enable
configure terminal
hostname ???
interface ???
 ip address ??? ???
 no shutdown
 exit
```

Ära unusta: `no shutdown` — IOL-XE pordid on vaikimisi **väljas**.

</details>

**SW1** — seadista hostname. Muud ei ole vaja, sest SW1 on tavaline L2 switch — ta lihtsalt edastab kaadreid, VLANe siin laboris ei kasuta.

**DNSMASQ** (ava konsool: paremkliki → **Web console**) — see on Linux konteiner, mitte IOS. Anname talle IP-aadressi ja ütleme kuhu saata paketid mis ei ole lokaalses võrgus:

```
ip addr add 172.16.0.10/24 dev eth0
ip route add default via 172.16.0.1
```

Esimene rida annab konteinerile IP. Teine ütleb: "kõik mis ei ole `172.16.0.0/24` võrgus — saada `172.16.0.1` kaudu" (see on R1).

### 1.3 Kontrolli ühendust

Testi R1-lt:

```
ping 172.16.0.10
ping 10.10.X.1
```

Kõik peaksid töötama.

> ⚠️ PC1-l ja PC2-l **ei ole veel IP-aadressi** — nad saavad selle DHCP kaudu.

<details>
<summary>❓ Ping ei tööta? Proovi kõigepealt ise lahendada, siis kliki siia.</summary>

1. `show ip interface brief` R1-l — kas liides näitab `administratively down`? Kui jah → `no shutdown` on puudu. IOL-XE seadmetes on pordid **vaikimisi väljas**.
2. Kas kaablid on õigete portide vahel? Vaata topoloogiat uuesti.
3. Kas IP-aadress on õige? Kas asendasid `X` oma M-numbriga?
4. Kas DNSMASQ konteineris on IP seadistatud? Proovi konteineris `ip addr show eth0`.

</details>

---

## Osa 2 — Mõtle enne kui seadistad

**Ära veel midagi seadista!** Vasta kõigepealt nendele küsimustele (kirjuta vastused paberile või faili):

1. DHCP kasutab **broadcast** päringut (DHCPDISCOVER). Broadcast ei läbi ruuterit. Sinu DHCP server on **teises alamvõrgus** kui kliendid. Kuidas jõuab DHCP päring serverini?

2. Vaata topoloogiat. Mis liidesele R1-l pead seadistama `ip helper-address`? Miks just sellele?

3. dnsmasq peab teadma, **mis aadressivahemiku** klientidele jagada. Kas see vahemik on serveri alamvõrgu aadressid (172.16.0.x) või kliendi alamvõrgu aadressid (10.10.X.x)? Miks?

4. DHCP jagab lisaks IP-le ka muid parameetreid. Mis kaks olulist asja peaks klient DHCP-lt saama, et ta saaks a) internetti ja b) nimepäringuid teha?

5. DNS A-kirje seob **nime** IP-aadressiga. Kui tahad, et `server.labor.ee` viitaks aadressile `172.16.0.10` — mis tüüpi kirje see on ja kuhu see konfigureeritakse?

<details>
<summary>❓ Jäid hätta? Kliki vihjele.</summary>

Loe loengumaterjalina `dhcp_dns.md` — seal on DHCP relay ja DNS kirjete selgitus.

</details>

---

## Osa 3 — Seadista DHCP

### 3.1 dnsmasq DHCP konfiguratsioon

Ava DNSMASQ konsooli (paremkliki → **Web console**). Sul on vaja teha **kolm asja** dnsmasq-is:

1. ☐ Määra DHCP aadressivahemik **sinu kliendivõrgule** (`10.10.X.0/24`)
2. ☐ Määra DHCP option: default gateway
3. ☐ Määra DHCP option: DNS server

Konfiguratsioonifail asub: `/etc/dnsmasq.d/lab.conf`

Muuda faili:

```
vi /etc/dnsmasq.d/lab.conf
```

<details>
<summary>❓ Ei oska vi-d kasutada?</summary>

Otsi konkreetset: `how to save and exit vi` või `how to edit file in vi`.

</details>

**Käske siin ei ole.** Kasuta loengumaterjalina `dhcp_dns.md` — sealt leiad dnsmasq konfiguratsioonisüntaksi.

<details>
<summary>❓ Ei saa dnsmasq süntaksist aru? Kliki vihjele.</summary>

Vaata `dhcp_dns.md` — seal on iga võtmesõna selgitatud koos näidetega.

</details>

Pärast muudatusi laadi dnsmasq uuesti:

```
kill -HUP $(pidof dnsmasq)
```

### 3.2 DHCP relay ruuteril

R1-l on vaja teha **üks asi**:

1. ☐ Seadista `ip helper-address` õigele liidesele

**Käsku siin ei ole.** Mõtle: mis liidesele ja mis aadressiga?

<details>
<summary>❓ Ei tea kuhu helper-address panna? Mõtle kõigepealt, siis kliki.</summary>

`ip helper-address` pannakse sellele liidesele, kust **DHCP broadcast sisse tuleb** — ehk klientide poolsele liidesele. Aadress on DHCP serveri IP.

</details>

### 3.3 Testi DHCP

PC1-l:

```
ip dhcp
```

PC2-l:

```
ip dhcp
```

Mõlemad peaksid saama IP-aadressi vahemikust `10.10.X.100` – `10.10.X.200`.

Kontrolli PC1-l: `show ip` — kas IP, mask, gateway ja DNS on kõik olemas?

<details>
<summary>❓ DHCP ei tööta? Proovi ise lahendada, siis kliki siia.</summary>

- Kas dnsmasq konfiguratsioon on korrektne? (Vaata uuesti `dhcp_dns.md`)
- Kas `ip helper-address` on õigel liidesel ja õige aadressiga?
- Kas R1-lt saad pingida DNSMASQ-i? (`ping 172.16.0.10`)
- Kas dnsmasq jookseb? (Konsoolis: `pidof dnsmasq` — peaks näitama numbrit)
- Kas R1 liidesed on üleval? `show ip interface brief` — kui `administratively down`, siis `no shutdown`.

</details>

---

## Osa 4 — Seadista DNS

### 4.1 Lisa DNS-kirjed dnsmasq-i

Ava uuesti DNSMASQ konsool. Lisa konfiguratsiooni DNS-kirjed:

1. ☐ `server.labor.ee` → `172.16.0.10`
2. ☐ `gw.labor.ee` → `172.16.0.1`
3. ☐ `gw-lan.labor.ee` → `10.10.X.1`

**Käske siin ei ole.** Kasuta loengumaterjalina `dhcp_dns.md` — sealt leiad `address=` süntaksi.

<details>
<summary>❓ Kuidas DNS-kirjeid lisada? Proovi ise, siis kliki.</summary>

Vaata `dhcp_dns.md` — osa "DNS konfiguratsioon".

</details>

Laadi dnsmasq uuesti: `kill -HUP $(pidof dnsmasq)`

### 4.2 Testi DNS R1-lt

Seadista R1, et ta kasutaks DNSMASQ-i DNS-serverina:

```
configure terminal
ip name-server 172.16.0.10
ip domain-lookup
end
```

Nüüd testi:

```
ping server.labor.ee
ping gw.labor.ee
```

Kas nimeresolutsioon töötab? Peaksid nägema IP-aadressi, mitte `% Unrecognized host`.

<details>
<summary>❓ DNS ei resolvi? Proovi ise, siis kliki siia.</summary>

- Kas dnsmasq-i DNS-kirjed on õigesti kirjutatud? Vaata `cat /etc/dnsmasq.d/lab.conf`
- Kas dnsmasq laaditi uuesti? `kill -HUP $(pidof dnsmasq)`
- Kas R1-l on `ip domain-lookup` sees? (Vaikimisi on, aga `no ip domain-lookup` lülitab välja)
- Kas R1-lt saad pingida DNSMASQ-i IP-ga? `ping 172.16.0.10`

</details>

### 4.3 Testi DNS klientidelt

Kui sa DHCP konfiguratsioonis määrasid DNS serveri õigesti, peaks ka PC1 ja PC2 olema DNS infoga varustatud.

Kontrolli PC1-l: `show ip` — kas DNS server on näha?

> ⚠️ VPCS ei toeta täielikku DNS resolutsiooni. DNS testimiseks kasuta R1-d. VPCS-i puhul piisab, et `show ip` näitab õiget DNS aadressi.

---

## Osa 5 — Analüüsi tulemusi

### 5.1 DHCP leased

DNSMASQ konsoolis:

```
cat /var/lib/misc/dnsmasq.leases
```

### Täida tabel:

| Aegumisaeg (timestamp) | MAC aadress | IP aadress | Hostname |
|------------------------|-------------|------------|----------|
| | | | |
| | | | |

### Vasta küsimustele:

1. Mitu lease'i on tabelis? Kas see vastab klientide arvule?

2. Mis on lease'i **aegumisaeg** (timestamp)? Miks DHCP üldse aegu kasutab — miks mitte anda aadress igaveseks?

3. Vaata MAC aadresse. Kas need vastavad sinu PC1 ja PC2 MAC-idele? (Kontrolli VPCS-is: `show ip`)

4. R1-l: `show ip interface brief` — kas R1 sai samuti DHCP aadressi? Miks mitte?

### 5.2 DNS päringud

R1-l:

```
ping server.labor.ee
```

Vaata väljundit. Mis IP-aadressiks resolvis? Kas see vastab sinu DNS-kirjele?

Proovi ka nime, mida sa **ei lisanud** dnsmasq-i (nt `ping test.labor.ee`). Mis juhtub?

---

## Osa 6 — Riku ja paranda

### Harjutus A — Eemalda helper-address

R1-l:

```
configure terminal
interface e0/0
 no ip helper-address 172.16.0.10
 exit
end
```

Nüüd PC1-l: `ip dhcp`

- Mis juhtub? Saab PC1 aadressi?
- Miks?
- Mida see ütleb DHCP relay'i kohta?

**Pane tagasi!**

### Harjutus B — Vale aadressivahemik

Muuda dnsmasq konfiguratsioonis DHCP vahemik serveri alamvõrgu aadressideks (nt `172.16.0.100,172.16.0.200`). Laadi uuesti.

PC1-l: `clear ip` ja siis `ip dhcp`

- Mis aadressi PC1 saab?
- Kas PC1 saab pingida R1-d (`10.10.X.1`)? Miks?

**Paranda tagasi!**

### Harjutus C — Kustuta DNS-kirje

Eemalda dnsmasq konfiguratsioonist `server.labor.ee` kirje. Laadi uuesti.

R1-l:

```
ping server.labor.ee
```

- Mis juhtub?
- Kas IP-aadressiga (`ping 172.16.0.10`) ikka töötab?
- Mis see ütleb DNS-i ja IP-ühenduse vahe kohta?

**Paranda tagasi!**

### Arutelu paarilisega

Enne esitamist — seleta oma paarilisele:
- Mis vahe on DHCP relay'il ja DHCP serveril?
- Miks DNS kirje kadumine ei katkesta IP-ühendust?

Kui paarilline ei nõustu — arutage koos välja.

---

## Esita tulemus

Igaüks esitab **oma** projektist. Mõlemad paarilised peavad esitama — muidu kumbki ei saa arvestust.

Tee screenshot (või kopeeri tekst) järgmistest:

1. `cat /var/lib/misc/dnsmasq.leases` (DNSMASQ konsoolis)
2. `show ip` mõlemal VPCS-il (PC1 ja PC2) — peab näitama DHCP-lt saadud IP-d
3. `ping server.labor.ee` (R1-lt) — peab resolvima ja pingima
4. `cat /etc/dnsmasq.d/lab.conf` (DNSMASQ konsoolis) — sinu konfiguratsioon
5. `show running-config` (R1-l) — peab sisaldama `ip helper-address`
6. Osa 2 küsimuste vastused

---

## Salvesta

R1 ja SW1:

```
copy running-config startup-config
```