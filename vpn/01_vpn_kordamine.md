# VPN — esimene päev: kus see üldse pildilt välja kasvab

**Kursus:** Arvutivõrgud II
**Päev:** 1/3 (KORDAMINE — meeldetuletus + suur pilt)
**Sinu eelteadmised:** VLAN, inter-VLAN routing, DHCP, NAT, ACL, OSPF. Krüpteerimisest tead nime järgi, et AES ja RSA on olemas.

## 🎯 Mida sa pärast täna oskad

Pärast seda päeva oskad:

- selgitada, miks me ei saa kaht filiaali lihtsalt VLANiga ühendada
- põhjendada, miks oma kaabli paigaldamine kahe linna vahele pole tavaliselt mõistlik
- näidata vähemalt kolme **erinevat olukorda**, kus VPN-i kasutatakse, ja seletada, mille poolest need erinevad
- defineerida VPN-i nii, et iga sõna sees (virtual, private, network) ka midagi tähendab
- eristada **site-to-site** ja **remote access** VPN-i
- nimetada kolm asja, mida me VPN-iga tahame kaitsta (CIA)

---

## 0. Kuhu me eelmisel päeval jõudsime

Eile lõpetasime OSPF-iga. Kolm ruuterit, naabrid leidsid üksteist, marsruudid täitusid automaatselt, ping läks läbi. Hea uudis: routing **töötab**. Sinu kontoris ükskõik mis võrgustruktuur ehita — ruuter teab, kuhu pakk saata.

Aga kogu see lugu eeldas ühte asja: **võrk oli sinu**. Sinu kaablid, sinu ruuterid, sinu IP-aadressid. OSPF rääkis sinu ruuterite vahel ja keegi väljast ei kuulanud.

Täna küsime: **mis siis, kui võrk pole sinu?**

Konkreetne probleem: sul on firma, mille üks pool istub Tallinnas ja teine pool Tartus. Mõlemas otsas on oma sisevõrk. Vahel — `185 km` Eesti maad, mida sa ei oma. Tallinna inimene tahab Tartu failiserverit, Tartu inimene tahab Tallinna andmebaasi. Kuidas?

> ✏️ **Mõtle hetkeks paberile.** Enne kui me edasi läheme — proovi joonistada, **mis variandid** sul üldse oleksid kahe linna vahelise võrgu ehitamiseks. Mitte VPN-i — mis tahes lahendused. Kirjuta kolm tükki paberile.

---

## 1. Variant A — VLAN ei sobi

Sa võiksid mõelda: "VLAN-id me oskame! Paneme Tallinna ja Tartu sama VLANi. Kõik ühes loogilises võrgus, valmis."

**Aga see ei tööta.** Vaatame, miks.

VLAN on **logiline jaotus füüsilise võrgu sees**. Kaks olulist sõna: *logiline* ja *füüsilise*. Logiline tähendab, et VLAN-id ei ole päris kaabliga seotud — sama kaabli peal võivad mitu VLANi liikuda. Aga **füüsiline** osa on see, mille taga me täna oleme: VLANid liiguvad **trunk-pordi kaudu**, ja trunk-port on **kahe switchi vahel**. Need switchid peavad olema **omavahel kaabliga ühendatud**.

```
Tallinna kontor                    Tartu kontor
                                       
  SW1 ──trunk?── SW2     ←  185 km    ──     SW3 ──── PC
   │                                          │
   PC                                         PC
```

Trunk-i ei saa **üle interneti** vedada. ISP ei tee sinu Etherneti raameid edasi — ta tegeleb IP-pakkidega ja need on **võrgukihi** (Layer 3) asi, VLAN on **kanali kihi** (Layer 2) asi. Sinu VLAN-info läheks kaduma kohe esimese ISP-ruuteri taga.

Lisaks: isegi kui sa kuidagi suudaks Layer 2-trunki kahe linna vahel ehitada (mõned ISP-d pakuvad seda teenust — nimetatakse `Carrier Ethernet` või `Metro Ethernet`), oleks see ikkagi **avatud**. VLAN ei krüpteeri midagi. Iga raami sisu liiguks puhta tekstina.

> 💡 **Kokkuvõte:** VLAN töötab seal, kus sa **kontrollid kaableid**. Kui kaabel pole sinu — VLAN ei toimi.

---

## 2. Variant B — paneme oma kaabli

OK, kaabel on probleem. Aga me võiksime kaabli **päriselt paigaldada**. Telia või Elisa müüvad sulle **liisitud liini** (`leased line`) või **musta kiu** (`dark fiber`) — füüsiline kaabel Tallinna ja Tartu vahel, mida kasutab **ainult sinu firma**.

```
Tallinna kontor ════════════════ Tartu kontor
                  oma kaabel
              (rendib Telia)
```

**Plus:** see töötab. Kiirus on garanteeritud, latentsus on madal, pealtkuulajaid praktiliselt pole (raske ilma maad lahti kaevamata). Suurpangad, kindlustusfirmad, riigiasutused kasutavad seda kõige tundlikuma liikluse jaoks.

**Miinus:** hind. Liisitud liin Tallinn–Tartu maksab kuus tüüpiliselt mitu tuhat eurot. Kui sul on Tartus kaheksa inimest, on see lahendus **väga ebamõistlik**. Igale järgmisele filiaalile (Pärnu, Narva, Kuressaare) tuleb veel üks selline liin — või veel hullem, **kõigi vaheline pluss kõigi vaheline**, mis ruutkasvab.

> ✏️ **Aruta paariga.** Sul on viis filiaali (Tallinn, Tartu, Pärnu, Narva, Kuressaare) ja kõik peavad omavahel ühenduses olema. Kui igale paari vahele eraldi kaabel — **mitu kaablit kokku** vaja oleks? Vihje: arvuta, mitu paari saab teha viiest punktist.

> **Vastus:** 10 kaablit. (5×4 / 2 = 10.) Kümme liisitud liini × paar tuhat eurot kuus = kuu kohta üle 20 000 €. Aastas üle veerand miljoni. Ainult kaablite eest.

Liisitud liin on hea **kõige tähtsama liikluse jaoks** — pangaülekannete tagavaratee, riiklik infrastruktuur. Tavalisele firmale: **liiga kallis**.

---

## 3. Variant C — saadame lihtsalt üle interneti

OK, oma kaabel kallis. Aga internet on **odav** — Tallinnas on sul juba ISP-ühendus, Tartus ka. **Saadame paketid lihtsalt sealt läbi**.

```
Tallinn ──── ISP1 ──── INTERNET ──── ISP2 ──── Tartu
            (avalikud marsruuterid, kümned)
```

**Mis selle juures valesti läheb?** Vasta kolm asja:

1. ☐ ...

2. ☐ ...

3. ☐ ...

> 💡 **Lahti võtmiseks — proovi paarilisega 2 minutit.**

> **Vastused, mida sa peaksid saama (kui ei tulnud meelde, loe rahulikult uuesti läbi):**
>
> 1. **Keegi võib pealt kuulata.** Iga ISP-ruuter teel — sealhulgas teiste maade omad — näeb pakkide sisu. Kui sa saadad lepingu PDF, näeb see ruuter PDF-i. Kui sa saadad parooli — näeb parooli.
>
> 2. **Keegi võib paketid teel ära muuta.** Ülekannete summad, lepingute sisu, e-kirjad — kõik on muudetav, kui pealtkuulaja ka samal teel asub.
>
> 3. **Sa ei tea, kellega sa räägid.** Pakid lähevad Tallinnast välja, "kellelegi". Kuidas sa kindel oled, et teises otsas on tegelikult sinu Tartu ruuter, mitte keegi, kes mängib end sinu Tartu ruuteriks?
>
> 4. **Sinu sisevõrgu IP-aadressid ei marsruudita üle interneti.** Tartu ruuteri sees on `10.10.1.0/24`, Tallinnas `10.10.2.0/24`. Need on **privaatsed aadressid**, mida ISP-marsruuterid ei tunne. Pakk lihtsalt **kuhugi ei jõua**.

Probleem on **suurem kui ainult turvalisus**. Tavaline internet ka **füüsiliselt ei marsruudi** sinu sisevõrgu pakke kahe linna vahel.

---

## 4. Variant D — VPN ütleb "tee tunneli sisse"

Kõik kolm eelmist varianti olid puudulikud. VLAN ei töötanud, oma kaabel kallis, internet avatud ja ei marsruudi sisevõrgu aadresse. VPN võtab need probleemid ükshaaval lahendusse.

**Idee on lihtne — pane sinu pakid teise paki sisse.**

```
Mis päriselt ringi liigub internetis:

┌─────────────────────────────────────────────────────────┐
│  Uus IP-päis           │  Krüpteeritud sisu             │
│  (Tallinn → Tartu      │  ┌──────────────────────────┐  │
│   avalikud IP-d)       │  │ Sinu päris pakett        │  │
│                        │  │ (10.10.2.5 → 10.10.1.7)  │  │
│                        │  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
        ↑                              ↑
    ISP näeb seda                ISP-l pole sellele
    ja marsruudib                 ligipääsu — krüpteeritud
```

Välimine paki päis kannab **avalikku adresseerimist** — Tallinna ruuteri välismaa IP-d ja Tartu ruuteri välismaa IP-d. ISP marsruutimine töötab nagu tavaliselt, sest need on **avalikud aadressid, mida ISP tunneb**.

Aga **sees** on sinu päris pakett — sinu sisevõrgu IP-aadressidega, sinu rakenduse andmetega, kõik krüpteeritud. ISP näeb ainult ümbrise, mitte sisu.

Nimi **virtual private network** tähendab nüüd seda:

- **Virtual:** ei ole päris oma kaabel. Pakk jagab teed kümnete teiste ISP-klientidega.
- **Private:** sisu pole väljapaistav. Pealtkuulajal pole võtit, krüpteerimise tõttu loeb ta ainult juhuslikku müra.
- **Network:** kogu IP-liiklus läheb läbi (mitte ainult üks rakendus nagu HTTPS-i puhul).

> 💡 **Kokkuvõte siin:** VPN võtab **kaks asja korraga** — tunneerimise (pakk paki sees) **ja** krüpteerimise. Eraldi kumbki ei lahenda kogu probleemi. Tunneerimine ilma krüpteerimiseta annab ühenduse (sinu sisevõrgu aadressid jõuavad kohale), aga ei kaitse sisu. Krüpteerimine ilma tunneerimiseta kaitseb sisu, aga ei lahenda marsruutimist.

---

## 5. Aga see ei ole **ainult** Tartu–Tallinn

Siiani võiksid mõelda, et VPN on ainult kahe kontori ühendamiseks. **See on kõige tavalisem juhtum, aga mitte ainus.**

Vaatame veel kolme reaalset kohta, kus VPN on igapäevatöös.

### 5.1 Serverite vahel — datakeskused ja pilved

Bolt, Wise, Pipedrive — kõigil neil on **mitu datakeskust**. Mõned omad, mõned AWS-i / Azure'i / Google Cloudi peal. Üks andmebaasi tagavarakoopia istub Eestis, teine Iirimaal või Frankfurdis.

Need datakeskused **peavad omavahel rääkima**. Andmed sünkroniseeruvad: tehing, mis tehakse Eesti serveris, peab ka Frankfurdi serveris välja paistma — muidu kaotad transaktsioone.

```
Eesti datakeskus ═══ VPN (IPSec) ═══ Frankfurdi datakeskus
                                                ║
                                                VPN
                                                ║
                                       Iirimaa datakeskus
```

Selle vahelise liikluse kaitsmiseks kasutatakse **serverite-vahelist VPN-i**. Tavaliselt site-to-site IPSec — täpselt nagu kahe kontoriruuteri vahel, aga otspunktid on **datakeskuste lüüsid** (firewall'id, koormusjaoturid). Kasutaja sellest mitte midagi ei tea.

> ✏️ **Mõtle:** mis võiks juhtuda, kui üks selline vahelink **ei oleks** krüpteeritud? Mis liiklus seal voolab? Mis info võib lekkida?

Vastus: kõik. Klientide makseinfo, salasõnad, isiklik info, sisemine API-suhtlus. Andmekaitse seaduste järgi (`GDPR`) **see oleks rikkumine** — ja Bolt / Wise tasemel firma puhul tähendaks see kümnetes miljonites trahvi.

### 5.2 Teenusepakkujate vahel — pangandus ja MPLS VPN

Pangad ei saada raha lihtsalt üle interneti. Kui sa teed Coopist ülekande SEB-i, **ei lähe see Coopi serverist SEB serverisse otse**. Vahel on **rahvusvaheline pangandusvõrk**, mille üks tuntumaid on SWIFT.

SWIFT-võrk on ise pankadevaheline **eraldi võrk**, kuhu pankadel on eriühendused — kombinatsioon VPN-idest, sertifikaatidest, dedikeeritud liinidest. Kui üks pank tahab teisele teateid saata, läheb see SWIFTi võrgu kaudu, mis ise on aastakümneid ehitatud turvalisuse-keskse infrastruktuurina.

Operaatorid ise (Telia, Elisa) pakuvad samuti **MPLS VPN** teenust — ärikliendid saavad teenusepakkujalt "virtuaalse erasalvõrgu", mis paistab oma sisemise võrguna, aga jookseb teenusepakkuja avalikul infrastruktuuril. See on **tüüpiline keskmise-suure ettevõtte lahendus** filiaalide ühendamiseks ilma, et iga filiaal peaks oma VPN-tunnelit konfigureerima.

> 💡 **Mida pidada meeles:** VPN ei ole alati "ühe firma asi". On olemas **operaatori VPN-id**, kus VPN-i teenuse pakubki sulle ISP. Sina ei pea ise krüpteerimisega tegelema — see käib teenuse hinnas.

### 5.3 Üksiku kasutaja jaoks — kodutöö

Sina istud kodus, sülearvuti on 4G-modemiga internetis. Sul on vaja ettevõtte failiserverisse. Kogu eelmine struktuur — ruuterite-vaheline VPN — ei sobi siia, sest sinu kodu pole "ettevõtte teine kontor".

Selle jaoks on **remote access VPN**. Sa käivitad sülearvutil kliendi (Cisco AnyConnect, OpenVPN, WireGuard, oma firma oma), logid sisse oma kontoriga, ja **sinu sülearvuti hakkab käituma nagu ta oleks kontori sisevõrgus**. Sa näed sisemisi servereid, saad SSH-ga, saad failiserveri lahti — kõik täpselt nagu töökohas.

```
Sinu kodus                                 Ettevõte
sülearvuti  ──── 4G ──── INTERNET ──── VPN-server ──── sisevõrk
   │                                                    │
   └─── tunnel (krüpteeritud) ───────────────────────────┘
```

Kogu sinu kodus tehtud liiklus, mis kontori võrku läheb, käib tuneli kaudu. ISP näeb ainult, et sinu sülearvuti räägib **ühe IP-aadressiga ettevõtte servereil** — ei näe, et sa avad failiserverit, ei näe, mis sa SSH-s teed.

> 💡 **Eristamiseks:**
>
> | Tüüp | Otspunktid | Kasutuskoht |
> |---|---|---|
> | **Site-to-site VPN** | Kaks ruuterit (kaks võrku) | Tallinn↔Tartu, Eesti↔Frankfurt datakeskused |
> | **Remote access VPN** | Üks arvuti ↔ üks server | Sina kodus → ettevõtte sisevõrk |

Mõlemad kasutavad sageli sama tehnoloogiat (IPSec, või tänapäeval WireGuard / OpenVPN), aga **konfiguratsioon ja kasutaja kogemus on erinevad**.

---

## 6. Aga mida me täpselt kaitseme — CIA

VPN ei ole maagia. Ta lubab täpselt **kolme asja**, mitte rohkem. Nendele otsime krüpteerimisest lahendust. Inglise keeles **CIA** — Confidentiality, Integrity, Authentication.

### 6.1 Konfidentsiaalsus (Confidentiality)

**Sõnum, mida sa saadad, ei ole loetav kellelegi peale saaja.**

Krüpteerimise abil teisendatakse sinu sisu (lepingu PDF, e-kiri, parool) nõnda, et see näeb välja juhuslik müra. Ainult kellel on **õige võti**, saab sõnumi tagasi originaaliks. Algoritmid: AES (kõikjal levinud), ChaCha20 (uus, mobiilis populaarne).

> ✏️ **Mõtle:** mis siis, kui sa sõnumi krüpteerid, aga keegi püüab kõik su krüpteeritud paketid kinni — terabaitide kaupa? Kas ta saab kunagi võtme kätte?
>
> Vastus: tänapäeva algoritmidega (AES-256), **ei**. Isegi maailma kõige võimsamad superarvutid vajaksid universumi vanusest pikemat aega ühe võtme leidmiseks. **Krüpteerimine on praktiliselt murdmatu, kui võti on piisavalt pikk.**

### 6.2 Terviklikkus (Integrity)

**Sõnum, mis vastu võetakse, on sama nagu sõnum, mis saadeti — keegi pole teda teel muutnud.**

Lihtne näide, miks see tähtis: panga ülekanne 100 € → muudetakse teel 10 000 € peale, krüpteerimisega kasvõi. Saaja näeb 10 000 €-t. Sa saadad lepingu ühe versiooniga, saaja saab teise. Vaja on viisi, kuidas **kontrollida, et keegi pole bitti muutnud**.

Lahendus: **hash** (räsi). Saatja arvutab paki sisust kindla pikkusega "sõrmejälje" (algoritmid: SHA-256, SHA-1, MD5 — viimased kaks vananenud). Saatja paneb selle paki külge. Saaja arvutab uuesti — kui sõrmejäljed kattuvad, on sisu sama. Kui erinevad, **keegi muutis**.

### 6.3 Autentimine (Authentication)

**Sa tead, kellega sa päriselt räägid — see pole keegi, kes mängib end Tartu ruuteriks.**

Krüpteerimine ilma autentimiseta on tühi: krüpteerid sõnumi "Tartu ruuterile", aga tegelikult on pealtkuulaja vahepeal end Tartu ruuteriks teinud. Sina krüpteerid talle võtmega, mille tema teleks tegi. Kogu kaitse on katki.

Autentimiseks on kaks moodust:

- **Pre-shared key** (eeljagatud võti) — mõlemal poolel on sama saladus, mille nad eelnevalt kokku leppisid. Lihtne, töötab, aga ei skaleeru — kümne filiaali jaoks pead kümme võtit haldama, ja kui üks lekib, peab kõik vahetama.
- **Sertifikaat** — iga osaline omab digitaalset isikutunnistust, mida on signinud usaldatud osapool (Certificate Authority). Sama loogika kui ID-kaardiga: keegi (riik) garanteerib, et see kaart kuulub sellele inimesele. Skaleerub paremini, aga vajab keerukamat infrastruktuuri.

> ✏️ **Aruta paariga:** kumb meetod sobib paremini kahe filiaaliga firmale? Kumb 200 töötajaga ettevõttele, kus pooled töötavad kodust? Põhjenda.

---

## 7. Krüpto kahekordne süsteem — kiirelt meenutamiseks

Sa oled krüpteerimisest kuulnud, kõik IT-õppes käivad selle juurest läbi. Kiire meenutus, sest järgmistel päevadel tugineme sellele.

**Sümmeetriline krüpteerimine** — sama võti krüpteerib ja dekrüpteerib. Mõlemal poolel peab see sama võti olema. **Kiire**, aga **võtme jagamise probleem**: kuidas mõlemale poolele võti turvaliselt jagada, kui kanal pole veel turvaline? Algoritmid: AES, ChaCha20.

**Asümmeetriline krüpteerimine** — kaks võtit ühel inimesel: **avalik** (saab igaüks) ja **privaatne** (hoiad endale). Mis on krüpteeritud avalikuga, dekrüpteerub ainult privaatsega. **Aeglane**, aga **võtmejagamise probleem lahendatud**. Algoritmid: RSA, ECC.

**Kuidas need koos töötavad VPN-is** (ja paljudes muudes kohtades — HTTPS, SSH):

```
1. Asümmeetriline  ──→  Kahepoolne autentimine + sümmeetrilise võtme jagamine
                        (kasutatakse Diffie-Hellman algoritmi)
                  
2. Sümmeetriline   ──→  Tegelik andmevoog (kõik su päris paketid)
                        kasutab kiiret sümmeetrilist krüpteerimist
```

Esimene osa on aeglane, aga **lühike** — paar paketti vahetatakse, võti tuletub. Teine osa on kiire, see kestab terve sessiooni jooksul.

> 💡 **Üks asi meeles pidada:** **asümmeetriline = võtmevahetuseks**, **sümmeetriline = andmete jaoks**. Kahe asja jaoks kaks erinevat tööriista, sest nad lahendavad erinevaid probleeme.

---

## 8. Tänase päeva kokkuvõte

**VLAN ei sobi.** VLAN töötab seal, kus sa kaableid kontrollid. Üle interneti ei lähe.

**Liisitud liin liiga kallis.** Kahe-kolme kontori jaoks võimalik, aga ei skaleeru, ja krüpteerimist ei pakuks nagunii.

**Lahtine internet — kaks probleemi.** Esiteks privaatsed IP-d ei marsruudita. Teiseks isegi kui marsruutuks, oleks kõik avatud — pealtkuulamine, muutmine, võltsimine.

**VPN = tunneerimine + krüpteerimine.** Pakk paki sisse (välimisel päisel on avalikud aadressid, sees on sinu päris pakett), kogu sees olev sisu krüpteeritud.

**Kaks põhilist kasutusjuhtu.** Site-to-site (kaks ruuterit, kaks võrku) ja remote access (üks arvuti, üks server).

**Kolm kohta veel, kus VPN-i kohtad.** Serverite vahel (datakeskused, pilve-pilve), teenusepakkujate vahel (SWIFT, MPLS VPN), üksikute kasutajate jaoks (remote access — kodutöö).

**Kolm asja, mida me kaitseme.** Konfidentsiaalsus (krüpteerimine), terviklus (hash), autentimine (pre-shared key või sertifikaat). Kõik kolm on vajalikud, ükski ei asenda teist.

**Asümmeetriline + sümmeetriline koos.** Asümmeetriline teeb võtmevahetuse, sümmeetriline teeb tegeliku andmevoo. Standardne muster mitte ainult VPN-is, vaid igal pool — HTTPS, SSH, kõik.

---

## Mida me **homme** vaatame

Täna oleme suure pildi laual — **mida me tahame ja miks**. Homme läheme **kuidas täpselt** osasse.

Päevakorras:

- **IPSec** kui kuldstandard
- **IKE Phase 1 ja Phase 2** — kuidas kaks ruuterit kokku lepivad
- **AH vs ESP** — kaks protokolli IPSec-i sees
- **Transport vs tunnel mode** — millal kumba kasutada
- **NAT ja IPSec probleem** — miks ei taha kokku saada
- **Firewall ja VPN samas seadmes** — kuidas need koos töötavad

**Kolmandal päeval** ehitame ise kahe ruuteri vahele site-to-site IPSec-tunneli GNS3-s.

---

## Allikad ja lugemine kodus

### Kui tahad rohkem näha enne homset

| Mis | URL | Kestus |
|---|---|---|
| Cloudflare — What is a VPN? | https://www.cloudflare.com/learning/access-management/what-is-a-vpn/ | 5 min lugeda |
| Cisco — VPN overview | https://www.cisco.com/c/en/us/products/security/vpn-endpoint-security-clients/what-is-vpn.html | 10 min lugeda |
| NetworkLessons — VPN intro | https://networklessons.com/cisco/ccna-routing-switching-200-125/introduction-to-vpns | 15 min lugeda |

### Lihtne mõtteharjutus

Loe läbi üks Eesti suurfirma turvalisuse-aruanne (`security report`) — näiteks Coopi, LHV või Eesti Energia avalik info infoturbe kohta. Pane tähele, kus seal **VPN-i sõna** mainitakse ja kuidas. Sa hakkad nüüd mustreid nägema.

---

*Järgmine fail: `02_vpn_ipsec_syvitsi.md` — IPSec arhitektuur ja IKE detailid.*
