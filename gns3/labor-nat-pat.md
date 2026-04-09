# Labor: NAT/PAT seadistamine

## Õpieesmärgid

Selle labori lõpuks sa:
- ehitad töötava võrgu, kus sisevõrgu seadmed pääsevad "internetti"
- seadistad PAT ruuteril **iseseisvalt**, kasutades loengumaterjalina `02_nat_seadistamine.md`
- analüüsid NAT tabelit ja selgitad, mis seal toimub
- leiad ja parandad NAT seadistuse vigu

---

## Taust

Sul on firma, kus on 2 arvutit. ISP andis sulle ühe avaliku aadressi `200.0.0.1`. Sinu sisevõrk kasutab `10.10.10.0/24`. Sa pead seadistama ruuteri nii, et mõlemad arvutid saaksid internetti — kasutades ühte avalikku aadressi.

---

## Osa 1 — Topoloogia ja põhiseadistus (25 min)

### 1.1 Ehita topoloogia

Logi GNS3 sisse ja ava oma projekt. Ehita järgmine võrk:

```
    [PC1] ──── e0/1 [SW1] e0/0 ──── e0/0 [R1] e0/1 ──── e0/0 [SERVER]
    VPCS              IOL XE L2           IOL XE L3           VPCS
```

| Seade | Template | Nimi |
|-------|----------|------|
| R1 | IOL XE L3 | R1-SINU-NIMI |
| SW1 | IOL XE L2 | SW1-SINU-NIMI |
| PC1 | VPCS | PC1 |
| PC2 | VPCS | PC2 |
| SERVER | VPCS | SERVER |

Kaablid:

| Seade A | Port | Seade B | Port |
|---------|------|---------|------|
| SW1 | e0/0 | R1 | e0/0 |
| SW1 | e0/1 | PC1 | eth0 |
| SW1 | e0/2 | PC2 | eth0 |
| R1 | e0/1 | SERVER | eth0 |

Käivita kõik seadmed ja oota ~60 sekundit.

### 1.2 IP seadistus

Need käsud on antud, sest IP seadistus pole selle labori teema — fookus on NATil.

**R1:**

```
enable
configure terminal
hostname R1-SINU-NIMI

interface e0/0
 ip address 10.10.10.1 255.255.255.0
 no shutdown
 exit

interface e0/1
 ip address 200.0.0.1 255.255.255.0
 no shutdown
 exit
end
```

**SW1:**

```
enable
configure terminal
hostname SW1-SINU-NIMI
end
```

**PC1:** `ip 10.10.10.10 255.255.255.0 10.10.10.1`

**PC2:** `ip 10.10.10.20 255.255.255.0 10.10.10.1`

**SERVER:** `ip 200.0.0.100 255.255.255.0 200.0.0.1`

### 1.3 Kontrolli ühendust

Testi PC1-lt:

```
ping 10.10.10.1
ping 200.0.0.100
```

Mõlemad peaksid töötama. Kui ei tööta — kontrolli kaableid ja IP-sid enne kui edasi lähed.

---

## Osa 2 — Mõtle enne kui seadistad (10 min)

**Ära veel midagi seadista!** Vasta kõigepealt nendele küsimustele (kirjuta vastused paberile või faili):

1. Praegu PC1 ping serverit töötab. Päris elus see nii ei oleks — miks?

2. Vaata oma topoloogiat. Mis liides R1-l on **inside** ja mis on **outside**? Miks?

3. NAT seadistamiseks on vaja ACL-i. Mis on ACL-i ülesanne NAT kontekstis — mida see määrab?

4. Mis vahe on `ip nat inside source list 1 pool NIMI` ja `ip nat inside source list 1 interface e0/1 overload`? Millal kasutad kumbagi?

5. Mis on `overload` võtmesõna mõte? Mis juhtuks ilma selleta?

> **Vihje:** Kui jääd hätta, loe `01_nat_sissejuhatus.md` ja `02_nat_seadistamine.md`.

---

## Osa 3 — Seadista PAT iseseisvalt (20 min)

Nüüd seadista R1-le PAT (NAT overload). Sul on vaja teha **neli asja**:

1. ☐ Märgi inside liides
2. ☐ Märgi outside liides
3. ☐ Loo ACL, mis lubab sisevõrgu aadressid
4. ☐ Seo kõik kokku PAT käsuga

**Käske siin ei ole.** Kasuta materjali `02_nat_seadistamine.md`, osa "3. PAT — Variant A". Sealt leiad kõik vajaliku.

Kui oled seadistanud, testi:

```
ping 200.0.0.100      (PC1-lt)
ping 200.0.0.100      (PC2-lt)
```

Kui ping ei tööta — ära küsi kohe õpetajalt! Vaata kõigepealt `02_nat_seadistamine.md` osa "Levinumad vead" ja proovi ise leida, mis valesti on.

---

## Osa 4 — Analüüsi NAT tabelit (15 min)

Kui PAT töötab, tee PC1-lt ja PC2-lt mõlemalt ping serverile. Siis R1-l:

```
show ip nat translations
```

### Täida tabel:

| Pro | Inside local | Inside global | Outside global |
|-----|-------------|---------------|----------------|
| | | | |
| | | | |

### Vasta küsimustele:

1. Mis aadress on "Inside local"? Miks just see?

2. Mis aadress on "Inside global"? Kust see tuli?

3. Inside global on mõlemal real sama. Mis eristab PC1 ja PC2 liiklust? (Vaata tähelepanelikult tervet rida!)

4. Jooksuta ka `show ip nat statistics`. Mitu aktiivset tõlget on? Mis liides on inside ja mis outside?

---

## Osa 5 — Riku ja paranda (15 min)

### Harjutus A — Eemalda overload

Eemalda oma PAT käsk ja pane tagasi **ilma** `overload` võtmesõnata. Tühjenda NAT tabel:

```
clear ip nat translation *
```

Nüüd pingige PC1 **ja** PC2 samal ajal serverit.

- Mis juhtub?
- Miks?
- Mis tüüpi NAT see nüüd on?

Pane `overload` tagasi.

### Harjutus B — Vaheta inside/outside

Muuda liidesed vastupidi — inside saab outside'iks ja vastupidi. Proovi pingida.

- Mis juhtub?
- Vaata `show ip nat statistics` — mis on inside ja mis outside?

Paranda tagasi!

---

## Esita tulemus

Tee screenshot (või kopeeri tekst) järgmistest:

1. `show ip nat translations` (pärast kui PC1 ja PC2 on mõlemad pinginud serverit)
2. `show ip nat statistics`
3. `show running-config` (R1-l) — terve konfiguratsioon
4. Osa 2 küsimuste vastused

---

## Salvesta

```
copy running-config startup-config
```

Mõlemas seadmes (R1 ja SW1).