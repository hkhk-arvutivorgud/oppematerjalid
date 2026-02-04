# 802.1Q ja Trunk pordid

## Probleem: kuidas saata mitut VLANi ühe kaabli kaudu?

Kui sul on kaks switchi ja mõlemal on mitu VLANi, kuidas nad omavahel suhtlevad?

![VLANid kahe switchi vahel](https://cdn.networklessons.com/wp-content/uploads/2014/07/8021q-trunk-example.png)

Ilma trunk'ita vajaksid **iga VLANi jaoks eraldi kaablit** - see pole praktiline!

---

## Lahendus: Trunk link

**Trunk** on ühendus, mis kannab mitut VLANi ühe kaabli kaudu.

Aga kuidas switch teab, millisesse VLANi kaader kuulub? Tavaline Ethernet kaader ei ütle midagi VLANi kohta!

---

## 802.1Q märgistamine (tagging)

**802.1Q** on tööstusstandard, mis lisab Ethernet kaadrisse **4-baidise märgendi (tag)**.

![802.1Q kaader](https://cdn.networklessons.com/wp-content/uploads/2014/07/8021q-frame-headers.png)

**Märgendi struktuur (4 baiti = 32 bitti):**

| Väli | Suurus | Selgitus |
|------|--------|----------|
| EtherType | 16 bitti | Alati 0x8100 (ütleb et see on 802.1Q kaader) |
| Priority | 3 bitti | QoS prioriteet (0-7), kõrgem = tähtsam |
| CFI | 1 bitt | Canonical Format Indicator (tavaliselt 0) |
| VLAN ID | 12 bitti | VLANi number (0-4095) |

**12 bitti VLAN ID = 2^12 = 4096 võimalikku VLANi** (tegelikult 1-4094 kasutatavad)

---

## Kaks trunk protokolli

| Protokoll | Tüüp | Märkus |
|-----------|------|--------|
| **802.1Q** | IEEE standard | Kõik tootjad toetavad, **KASUTA SEDA!** |
| **ISL** | Cisco proprietary | Vana, ainult Cisco, enam ei kasutata |

---

## Kuidas märgistamine töötab?

1. **Arvuti** saadab tavalise Ethernet kaadri (märgistamata)
2. **Switch** võtab kaadri vastu access pordist
3. Switch teab, et see port on VLAN 50-s
4. Kui kaader läheb **trunk porti**, lisab switch **802.1Q märgendi** (VLAN 50)
5. Teine switch saab kaadri, loeb märgendi, teab et see on VLAN 50
6. Switch **eemaldab märgendi** ja saadab kaadri õigesse access porti

**NB!** Lõppseadmed (arvutid) ei näe kunagi märgendeid - need on ainult switchide vahel!

---

## Native VLAN

**Native VLAN** on eriline - selle liiklust **EI märgistata**!

Vaikimisi on Native VLAN = VLAN 1

```
SW1#show interface fa0/14 trunk

Port        Mode         Encapsulation  Status        Native vlan
Fa0/14      on           802.1q         trunking      1
```

**Miks Native VLAN eksisteerib?**
- Vanade seadmete jaoks, mis ei mõista 802.1Q
- Haldusprotokollid (CDP, VTP, DTP) liiguvad native VLANis

**TURVAVIHJE:** Muuda Native VLAN ära VLAN 1 pealt! VLAN 1 on rünnakute sihtmärk (VLAN hopping).

---

## Trunk seadistamine

```
SW1(config)#interface fa0/14
SW1(config-if)#switchport trunk encapsulation dot1q
SW1(config-if)#switchport mode trunk
```

**NB!** Mõned uuemad switchid toetavad ainult 802.1Q ja ei vaja `encapsulation dot1q` käsku.

---

## Trunk kontrollimine

```
SW1#show interfaces fa0/14 trunk

Port        Mode         Encapsulation  Status        Native vlan
Fa0/14      on           802.1q         trunking      1

Port        Vlans allowed on trunk
Fa0/14      1-4094

Port        Vlans allowed and active in management domain
Fa0/14      1,50

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/14      1,50
```

**TÄHTIS:** `show vlan` näitab AINULT access porte! Trunk porte seal pole!

---

## Switchport režiimid

| Režiim | Selgitus |
|--------|----------|
| `access` | Alati access port (üks VLAN) |
| `trunk` | Alati trunk port (mitu VLANi) |
| `dynamic auto` | Eelistab access, aga läheb trunk'iks kui teine pool tahab |
| `dynamic desirable` | Aktiivselt proovib trunk'i teha |

**TURVAVIHJE:** Ära kasuta dynamic režiime tootmisvõrgus! Määra alati selgelt `trunk` või `access`.

```
SW1(config-if)#switchport nonegotiate
```

---

## Käskude kokkuvõte

| Käsk | Selgitus |
|------|----------|
| `switchport mode trunk` | Sea port trunk režiimi |
| `switchport trunk encapsulation dot1q` | Kasuta 802.1Q |
| `switchport trunk native vlan 99` | Muuda native VLAN |
| `switchport trunk allowed vlan 10,20,30` | Luba ainult teatud VLANid |
| `switchport trunk allowed vlan add 40` | Lisa VLAN lubatute hulka |
| `switchport nonegotiate` | Lülita DTP välja |
| `show interfaces trunk` | Näita trunk infot |
