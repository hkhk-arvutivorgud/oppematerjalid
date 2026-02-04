# VLAN - Miks ja millal?

## Reaalsed kasutusjuhud

### 1. Ettevõtte kontor

```
Probleem:
HR töötab konfidentsiaalsete andmetega (palgad, isikuandmed).
Sales töötaja laptop'is on viirus.
Ilma VLANita → viirus pääseb HR arvutitesse.

Lahendus:
VLAN 10 = HR (192.168.10.0/24)
VLAN 20 = Sales (192.168.20.0/24)
VLAN 30 = IT (192.168.30.0/24)

Tulemus:
Sales'i viirus EI saa otse HR võrku.
Liiklus käib läbi tulemüüri, kus on reeglid.
```

### 2. VoIP telefonid

```
Probleem:
IP-telefonid saadavad kõnesid üle võrgu KRÜPTEERIMATA.
Wiresharkiga saab kõnesid salvestada ja kuulata.

Lahendus:
VLAN 100 = Voice (telefonid)
VLAN 10 = Data (arvutid)

Tulemus:
Telefoniliiklus on eraldatud.
QoS saab Voice VLANile prioriteedi anda.
```

### 3. Külalisvõrk (Guest WiFi)

```
Probleem:
Külalised vajavad WiFi-t, aga EI TOHI näha sisevõrku.

Lahendus:
VLAN 99 = Guest
- Ainult interneti ligipääs
- Blokeeritud: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- Client isolation

Tulemus:
Külaline saab internetti.
Külaline EI näe printereid, servereid, faile.
```

### 4. IoT seadmed

```
Probleem:
Nutilambid, Alexa, kaamerad - halvad turvauuendused.
Häkitud seade = ligipääs KOGU võrgule.

Lahendus:
VLAN 50 = IoT
- IoT → Internet: ALLOW
- IoT → LAN: DENY
- LAN → IoT: ALLOW

Tulemus:
Häkitud kaamera EI pääse sinu arvutisse.
```

---

## Mida VLAN teeb vs EI tee

| VLAN TEEB | VLAN EI TEE |
|-----------|-------------|
| Eraldab broadcast domain'i | Ei ole firewall |
| Grupeerib loogiliselt | Ei krüpteeri liiklust |
| Võimaldab QoS | Ei takista routing'ut VLANide vahel |

**VLAN + Firewall = turvalisus**
**VLAN üksi = organiseeritus**

---

## VLAN hopping rünnak

```
Ründaja saadab double-tagged frame:
[Outer: VLAN 1][Inner: VLAN 10][Data]

Switch eemaldab outer tag'i → frame läheb VLAN 10

Kaitse: 
- Ära kasuta VLAN 1
- switchport trunk native vlan 999
- Luba trunk'il ainult vajalikud VLANid
```

---

## Parimad praktikad

```
1. Ära kasuta VLAN 1 millekski

2. Native VLAN = kasutamata VLAN
   switchport trunk native vlan 999

3. Trunk'il ainult vajalikud VLANid
   switchport trunk allowed vlan 10,20,30

4. Kasutamata pordid → "dead" VLANi
   switchport access vlan 666
   shutdown
```

### Numbrite konventsioon:

```
VLAN 1     = Default (ära kasuta!)
VLAN 10    = Management
VLAN 20    = Servers
VLAN 30    = Users
VLAN 40    = Voice
VLAN 50    = IoT
VLAN 99    = Guest
VLAN 999   = Native (trunk)
VLAN 666   = Dead end
```

---

## Kokkuvõte

VLANid on **organiseerimise** tööriist, mitte **turvalisuse** tööriist.

**Hea kasutus:** broadcast vähendamine, loogiline grupeerimine, QoS, alus firewall reeglitele

**Halb kasutus:** "Meil on VLAN, oleme turvalised" (ei ole!)
