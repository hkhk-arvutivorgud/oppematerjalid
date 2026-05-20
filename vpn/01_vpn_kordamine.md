# VPN — esimene päev: kust see vajadus tekib

**Kursus:** Arvutivõrgud II  
**Päev:** 1/3 (kordamine, meeldetuletus ja suur pilt)  
**Eelteadmised:** VLAN, inter-VLAN routing, DHCP, NAT, ACL, OSPF; krüpteerimisest tead, et olemas on AES ja RSA.

## Mida sa pärast tänast oskad

Pärast seda päeva oskad:

- selgitada, miks kahte filiaali ei saa lihtsalt VLAN-iga ühendada
- põhjendada, miks oma kaabli vedamine kahe linna vahele ei ole tavaliselt mõistlik
- tuua vähemalt kolm olukorda, kus VPN-i kasutatakse, ja selgitada nende erinevusi
- defineerida VPN-i nii, et sõnad *virtual*, *private* ja *network* ka päriselt midagi tähendavad
- eristada site-to-site ja remote access VPN-i
- nimetada kolm asja, mida me VPN-iga kaitsta tahame: konfidentsiaalsus, terviklus ja autentimine

---

## 0. Kus me pooleli jäime

Eelmisel korral lõpetasime OSPF-iga. Ruuterid leidsid naabrid üles, marsruudid tekkisid automaatselt ja ühendus töötas. See tähendas, et ruuter teadis, kuhu pakett edasi saata.

Sellel lahendusel oli aga üks oluline eeldus: võrk oli sinu kontrolli all. Kaablid, ruuterid ja aadressiruum kuulusid sulle.

Täna küsime: mis saab siis, kui võrk enam sinu oma ei ole?

Näide: ettevõtte üks kontor on Tallinnas ja teine Tartus. Mõlemas kohas on oma sisevõrk. Tallinna töötaja tahab kasutada Tartu failiserverit ja Tartu töötaja Tallinna andmebaasi. Kuidas need kaks võrku ühendada nii, et lahendus oleks nii toimiv kui ka turvaline?

> ✏️ Enne edasi lugemist pane kirja kolm võimalikku lahendust, kuidas kahte linna omavahel ühendada.

---

## 1. Variant A — VLAN ei lahenda seda

Esimene mõte võib olla lihtne: paneme mõlemad kontorid samasse VLAN-i. Siis oleksid nad justkui ühes loogilises võrgus.

Praktikas see nii ei tööta. VLAN on loogiline jaotus füüsilise võrgu sees. VLAN-id liiguvad trunk-ühenduse kaudu ning trunk eeldab, et switchid on omavahel füüsiliselt ühendatud.

```text
Tallinna kontor                    Tartu kontor

  SW1 ── trunk? ── SW2   ← 185 km →   SW3 ─── PC
   │                                      │
   PC                                     PC
```

Teenusepakkuja interneti kaudu sinu Etherneti kaadreid edasi ei saada. Internetis liiguvad IP-paketid, mitte sinu lokaalse võrgu Layer 2 raamid (frames). See tähendab, et tavalist VLAN-i ei saa lihtsalt kahe linna vahele „läbi interneti“ venitada.

Isegi kui teenusepakkuja pakuks Layer 2 ühendust, jääks teine probleem alles: VLAN ei krüpteeri midagi. Liiklus oleks küll ühendatud, aga mitte kaitstud.

**Järeldus:** VLAN sobib sinna, kus füüsiline ühendus on sinu kontrolli all. Kahe linna vahelise avaliku võrgu jaoks sellest ei piisa.

---

## 2. Variant B — veame oma kaabli

Teine mõte on kasutada päris ühendust kahe koha vahel. Selleks võib rentida liisitud liini või musta kiu.

```text
Tallinna kontor ═════════════════ Tartu kontor
                oma ühendus
```

Selle lahenduse pluss on lihtne: see töötab hästi. Kiirus on ette teada, viivitus tavaliselt väike ja ühendus ei jaga teed juhusliku internetiliiklusega.

Miinus on hind. Kui ettevõttel on ainult paar väikest filiaali, võib selline lahendus olla liiga kallis. Mida rohkem filiaale lisandub, seda keerulisemaks ja kallimaks kogu ühendusskeem muutub.

Kui sul on viis filiaali ja igaüks peab kõigiga eraldi ühendatud olema, siis läheb vaja 10 ühendust. See kasvab kiiresti ebamõistlikuks.

**Järeldus:** oma liin on tehniliselt hea, kuid enamasti liiga kallis ja halvasti skaleeruv.

---

## 3. Variant C — saadame liikluse lihtsalt üle interneti

Kolmas mõte tundub kõige odavam: Tallinnas on internet, Tartus on internet, saadame liikluse lihtsalt sealt kaudu.

```text
Tallinn ─ ISP ─ INTERNET ─ ISP ─ Tartu
```

Siin tekib korraga mitu probleemi.

- Keegi võib liiklust pealt kuulata.
- Keegi võib teel pakette muuta.
- Sa ei saa olla kindel, et teises otsas on päriselt õige seade.
- Privaatsed sisevõrgu aadressid ei marsruudi üle avaliku interneti.

See tähendab, et probleem ei ole ainult turvalisuses. Tavaline internet ei oska sinu sisevõrgu pakette niisama õigesse kohta viia.

**Järeldus:** internet üksi on odav, kuid ei lahenda ei marsruutimist ega turvalisust.

---

## 4. Variant D — VPN teeb tunneli

VPN lahendab eelmiste variantide peamised puudused. Idee on lihtne: sinu päris pakett pannakse teise paketi sisse.

```text
Internetis liigub selline pakett:

┌──────────────────────────────────────────────┐
│ Välimine IP-päis                             │
│ (avalikud aadressid)                         │
│                                              │
│ Krüpteeritud sisu                            │
│ ┌──────────────────────────────────────────┐ │
│ │ Sinu päris pakett                        │ │
│ │ (näiteks 10.10.2.5 → 10.10.1.7)          │ │
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

Välimine päis kasutab avalikke IP-aadresse, mida internet oskab marsruutida. Sees on sinu päris pakett koos sisevõrgu aadressidega. See sisemine osa on krüpteeritud.

Siit tuleb ka nimetus **virtual private network**.

- **Virtual** — sul ei ole päris oma kaablit.
- **Private** — sisu on kõrvalistele osapooltele loetamatu.
- **Network** — läbi tunneli saab liikuda kogu IP-liiklus, mitte ainult üks rakendus.

VPN koosneb tegelikult kahest ideest:

- **tunneleerimine** — üks pakett pannakse teise sisse
- **krüpteerimine** — sees olev sisu tehakse võõrale loetamatuks

Kui kasutad ainult tunneleerimist, saad küll ühenduse, kuid mitte kaitset. Kui kasutad ainult krüpteerimist, ei lahenda sa veel sisevõrgu aadresside marsruutimist. VPN vajab mõlemat.

---

## 5. Kus VPN-i kasutatakse

VPN ei ole ainult kahe kontori ühendamiseks. Tegelikult kohtab seda mitmes eri olukorras.

### 5.1 Kahe võrgu vahel

Kõige klassikalisem näide on kahe kontori või kahe andmekeskuse ühendamine. Seda nimetatakse **site-to-site VPN-iks**.

```text
Tallinn ═══ VPN ═══ Tartu
```

Selles mudelis on otspunktideks tavaliselt ruuterid või tulemüürid. Kasutaja ise ei pruugi aru saadagi, et ta liiklus läheb läbi VPN-i.

### 5.2 Kasutaja ja ettevõtte vahel

Kui töötaja on kodus või reisil, ei ole tema arvuti ettevõtte teine kontor. Siin kasutatakse **remote access VPN-i**.

```text
Sülearvuti ─ INTERNET ─ VPN-server ─ sisevõrk
```

Kasutaja käivitab VPN-kliendi, logib sisse ja tema arvuti saab turvalise ühenduse ettevõtte sisevõrguga.

### 5.3 Teenusepakkujate ja suurte süsteemide vahel

VPN-e kasutatakse ka andmekeskuste, pilveteenuste ja teenusepakkujate võrkude vahel. Mõnikord haldab VPN-i ettevõte ise, mõnikord pakub seda teenusena operaator.

**Järeldus:** sama põhimõte, aga erinevad kasutusstsenaariumid.

---

## 6. Mida me VPN-iga kaitseme

VPN ei ole lihtsalt „turvaline toru“. Tavaliselt räägitakse kolmest põhieesmärgist, mida lühendatakse tähtedega **CIA**.

### 6.1 Konfidentsiaalsus

Konfidentsiaalsus tähendab, et kõrvaline osapool ei saa sinu andmeid lugeda. Seda tagab krüpteerimine. Levinud algoritmid on näiteks AES ja ChaCha20.

### 6.2 Terviklus

Terviklus tähendab, et andmeid ei ole teel muudetud. Vastuvõtja peab saama kontrollida, et sõnum on sama, mis saatja teele pani.

### 6.3 Autentimine

Autentimine tähendab, et sa tead, kellega sa tegelikult suhtled. VPN ei tohi luua tunnelit vale osapoolega.

Selleks kasutatakse näiteks:

- eeljagatud võtit (pre-shared key)
- sertifikaate

**Järeldus:** ainult krüpteerimisest ei piisa. Vaja on ka kontrolli, et andmeid pole muudetud ja et teine osapool on päriselt õige.

---

## 7. Kuidas krüptograafia siin töötab

VPN-ides kasutatakse tavaliselt kahte tüüpi krüpteerimist koos.

### 7.1 Sümmeetriline krüpteerimine

Sümmeetrilise krüpteerimise puhul kasutatakse sama võtit nii krüpteerimiseks kui ka dekrüpteerimiseks. See on kiire ja sobib hästi suure andmemahu jaoks.

Näited: AES, ChaCha20.

### 7.2 Asümmeetriline krüpteerimine

Asümmeetrilise krüpteerimise puhul on kasutusel avalik ja privaatne võti. See on aeglasem, kuid sobib võtmevahetuseks ja autentimiseks.

Näited: RSA, ECC.

### 7.3 Miks neid koos kasutatakse

Tüüpiline loogika on järgmine:

1. Asümmeetrilise krüptograafia abil tuvastavad osapooled teineteist ja lepivad võtme kokku.
2. Edasine andmeliiklus krüpteeritakse kiire sümmeetrilise algoritmiga.

Seda sama põhimõtet kasutatakse ka mujal, näiteks HTTPS-is ja SSH-s.

---

## 8. Tänase päeva põhiidee

- VLAN ei lahenda kahe linna vahelist ühendust üle avaliku interneti.
- Oma liin töötab, kuid on sageli liiga kallis.
- Tavaline internet üksi ei taga ei turvalisust ega sisevõrgu aadresside toimivat edastust.
- VPN ühendab kaks vajalikku asja: tunneleerimise ja krüpteerimise.
- VPN-i kasutatakse nii võrkude vahel kui ka üksiku kasutaja ühendamiseks ettevõtte sisevõrku.
- Kaitsta tuleb kolme omadust: konfidentsiaalsust, terviklust ja autentimist.
