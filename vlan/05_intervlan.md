# Inter-VLAN marsruutimine

## Probleem: VLANid ei räägi omavahel

Eelmises peatükis õppisime, et VLAN eraldab võrgu loogilisteks osadeks. HR osakond ühes VLANis, Sales teises, IT kolmandas. See on hea - broadcast'id ei levi enam üle terve võrgu ja meil on parem kontroll selle üle, kes kellega suhelda saab.

Aga nüüd tekib probleem.

Kujuta ette, et HR osakonna töötaja peab saatma faili Sales osakonna töötajale. HR on VLAN 10-s, Sales on VLAN 20-s. Mis juhtub kui HR arvuti proovib Sales arvutit pingida?

**Mitte midagi. Pakett ei lähe läbi.**

Miks? Sest VLANid on erinevad broadcast domain'id. Teisisõnu - need on täiesti erinevad võrgud. Ja mis seade ühendab erinevaid võrke? **Ruuter.** Ilma ruuterita ei saa erinevad VLANid omavahel rääkida.

See on nagu kaks eraldi hoonet ilma ühenduseta - inimesed mõlemas hoones eksisteerivad, aga nad ei saa üksteise juurde minna.

---

## Lahendus 1: Eraldi pordid (ära kasuta!)

Kõige lihtsam lahendus oleks ühendada iga VLAN eraldi kaabliga ruuteri eraldi porti. Kui sul on kolm VLANi, siis kolm kaablit ja kolm ruuteri porti.

See töötab, aga on **halb lahendus**:
- Ruuteritel pole tavaliselt palju Ethernet porte (4-8 tk)
- Kui sul on 10 VLANi, siis juba ei jätku
- Kaablite raiskamine
- Uue VLANi lisamisel pead uue kaabli tõmbama

Seda lahendust tänapäeval praktiliselt ei kasutata.

---

## Lahendus 2: Router-on-a-Stick

Parem lahendus on **Router-on-a-Stick**. Nimi kõlab naljakalt, aga kirjeldab hästi mida me teeme: ruuter istub "kepi otsas" ehk ühe ainsa lingi otsas.

### Kuidas see töötab?

Ruuteri ja switchi vahel on **üks trunk link**. See trunk kannab kõiki VLANe. Ruuteri poolel loome **subinterface'id** - virtuaalsed liidesed, üks iga VLANi jaoks.

```
         Trunk (802.1Q)
              │
    ┌─────────┴─────────┐
    │      Router       │
    │                   │
    │  G0/0 (no IP!)    │
    │    ├── G0/0.10 ──→ VLAN 10 → 192.168.10.1/24
    │    ├── G0/0.20 ──→ VLAN 20 → 192.168.20.1/24
    │    └── G0/0.30 ──→ VLAN 30 → 192.168.30.1/24
    │                   │
    └───────────────────┘
```

Vaatame samm-sammult, mis juhtub kui HR arvuti (VLAN 10, IP 192.168.10.10) tahab saata paketti Sales arvutile (VLAN 20, IP 192.168.20.10):

1. HR arvuti näeb, et sihtkoht 192.168.20.10 on **teises võrgus**, seega saadab paketi oma gateway'le (192.168.10.1)

2. Switch saadab paketi **trunk linki** kaudu ruuterisse. Pakett on märgistatud **VLAN 10 tag'iga**

3. Ruuter võtab paketi vastu **subinterface G0/0.10** kaudu (sest see kuulab VLAN 10 liiklust)

4. Ruuter vaatab marsruutimistabelit ja näeb, et 192.168.20.0/24 on ühendatud läbi **G0/0.20**

5. Ruuter saadab paketi välja läbi G0/0.20 - pakett saab **VLAN 20 tag'i**

6. Switch saab paketi trunk lingilt, näeb VLAN 20 tag'i ja saadab paketi õigesse access porti

See kõik toimub millisekunditega. Kasutaja ei märkagi, et pakett käis ruuteris ära.

### Seadistamine

**Switchi poolel** - tavaline trunk:
```
Switch(config)# interface G0/1
Switch(config-if)# switchport trunk encapsulation dot1q
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
```

**Ruuteri poolel** - kõigepealt aktiveeri füüsiline port (ilma IP-ta!):
```
Router(config)# interface G0/0
Router(config-if)# no shutdown
```

Siis loo subinterface'id:
```
Router(config)# interface G0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.10.1 255.255.255.0

Router(config)# interface G0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.20.1 255.255.255.0

Router(config)# interface G0/0.30
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 192.168.30.1 255.255.255.0
```

**Mida need käsud teevad:**
- `G0/0.10` - subinterface'i nimi (.10 on lihtsalt number, mõistlik kasutada sama mis VLAN ID)
- `encapsulation dot1Q 10` - seob selle subinterface'i VLAN 10-ga
- `ip address` - see saab selle VLANi gateway'ks

### Kontrollimine

```
Router# show ip interface brief

Interface              IP-Address      OK? Method Status Protocol
GigabitEthernet0/0     unassigned      YES unset  up     up
GigabitEthernet0/0.10  192.168.10.1    YES manual up     up
GigabitEthernet0/0.20  192.168.20.1    YES manual up     up
GigabitEthernet0/0.30  192.168.30.1    YES manual up     up
```

```
Router# show ip route

C    192.168.10.0/24 is directly connected, GigabitEthernet0/0.10
C    192.168.20.0/24 is directly connected, GigabitEthernet0/0.20
C    192.168.30.0/24 is directly connected, GigabitEthernet0/0.30
```

### Router-on-a-Stick puudus

Router-on-a-Stick töötab hästi väikestes võrkudes. Aga tal on üks suur puudus: **kõik VLANidevaheline liiklus käib läbi ühe trunk lingi**.

Kujuta ette kontorit, kus on 100 inimest - 50 HR-is, 50 Sales-is. Nad kõik saadavad üksteisele faile kogu aeg. Kõik need paketid peavad minema läbi ühe GigabitEthernet lingi ruuterisse ja tagasi. See link muutub **pudelikaelaks**.

Väikeses võrgus (kuni ~20 inimest) on Router-on-a-Stick suurepärane. Suures ettevõttes on vaja midagi võimsamat.

---

## Lahendus 3: L3 Switch (SVI)

Suure võrgu lahendus on **Layer 3 switch** - switch, mis oskab ka marsruutida. Sisuliselt ruuter ja switch ühes karbis.

Layer 3 switchil kasutame **SVI-sid (Switch Virtual Interface)**. SVI on virtuaalne liides, mis esindab tervet VLANi.

```
    ┌─────────────────────────┐
    │      L3 Switch          │
    │                         │
    │  VLAN 10 ← SVI → 192.168.10.1/24
    │  VLAN 20 ← SVI → 192.168.20.1/24
    │  VLAN 30 ← SVI → 192.168.30.1/24
    │                         │
    │  Fa0/1 ──► VLAN 10      │
    │  Fa0/2 ──► VLAN 20      │
    │  Fa0/3 ──► VLAN 30      │
    └─────────────────────────┘
```

### Miks L3 switch on parem?

| Router-on-a-Stick | L3 Switch |
|-------------------|-----------|
| Trunk link = pudelikael | Wire-speed routing |
| Eraldi ruuter + switch | Kõik ühes seadmes |
| Odavam | Kallim |
| Väike võrk, labor | Enterprise, andmekeskus |

L3 switchil **pole pudelikaela** - marsruutimine toimub switchi sees, riistvara tasemel, väga kiiresti.

### Seadistamine

Kõigepealt lülita routing sisse:
```
Switch(config)# ip routing
```

**See käsk on kriitiline!** Ilma selleta switch ei marsruudi, isegi kui SVI-del on IP-aadressid.

Siis loo SVI-d:
```
Switch(config)# interface vlan 10
Switch(config-if)# ip address 192.168.10.1 255.255.255.0
Switch(config-if)# no shutdown

Switch(config)# interface vlan 20
Switch(config-if)# ip address 192.168.20.1 255.255.255.0
Switch(config-if)# no shutdown

Switch(config)# interface vlan 30
Switch(config-if)# ip address 192.168.30.1 255.255.255.0
Switch(config-if)# no shutdown
```

**NB!** VLAN peab olemas olema ja vähemalt üks port peab olema selles VLANis, muidu SVI ei tule üles.

### Kontrollimine

```
Switch# show ip route

C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
C    192.168.30.0/24 is directly connected, Vlan30
```

---

## Tõrkeotsing

Kui VLANidevaheline suhtlus ei tööta, kontrolli järjest:

**1. Kas PC saab oma gateway'ni?**
```
PC> ping 192.168.10.1
```
Kui ei saa, siis probleem on VLANi sees, mitte VLANide vahel. Kontrolli kas port on õiges VLANis.

**2. Kas trunk töötab?**
```
Switch# show interfaces trunk
```
Kontrolli kas VLAN on trunk'il lubatud.

**3. Kas subinterface/SVI on üleval?**
```
Router# show ip interface brief
```
Peab olema `up/up`. Kui on `down`, kontrolli kas füüsiline port on `no shutdown` ja kas VLAN on trunk'il lubatud.

**4. L3 switchil: kas `ip routing` on sees?**
```
Switch# show ip route
```
Kui tabel on tühi, siis routing pole sisse lülitatud.

**5. Kas PC gateway on õige?**
Kui PC IP on 192.168.10.10 aga gateway on 192.168.20.1 - ei tööta.

---

## Kokkuvõte

| Meetod | Millal kasutada |
|--------|-----------------|
| Router-on-a-Stick | Väike võrk, labor, odav lahendus |
| L3 Switch (SVI) | Enterprise, andmekeskus, suur liiklus |

Mõlemal juhul on põhimõte sama:
- Iga VLAN vajab oma gateway IP-aadressi
- Routing toimub Layer 3 seadmes
- PC-de gateway peab olema õigesti seadistatud

---

## Käskude kokkuvõte

### Router-on-a-Stick
```
interface G0/0
  no shutdown

interface G0/0.10
  encapsulation dot1Q 10
  ip address 192.168.10.1 255.255.255.0
```

### L3 Switch (SVI)
```
ip routing

interface vlan 10
  ip address 192.168.10.1 255.255.255.0
  no shutdown
```
