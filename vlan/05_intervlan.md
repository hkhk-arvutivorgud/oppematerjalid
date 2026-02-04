# Inter-VLAN marsruutimine

## Probleem

VLANid eraldavad võrgu - see on hea! Aga...

```
VLAN 10 (HR)         VLAN 20 (Sales)
192.168.10.0/24      192.168.20.0/24
     │                    │
     └────── Switch ──────┘
     
PC1 (VLAN 10) ping PC2 (VLAN 20) = ???
```

**Vastus: EI TÖÖTA!**

VLANid on erinevad broadcast domain'id = erinevad võrgud. Erinevate võrkude vahel suhtlemiseks on vaja **ruuterit**.

---

## Kolm lahendust

| Lahendus | Kirjeldus | Millal kasutada |
|----------|-----------|-----------------|
| **Eraldi pordid** | Iga VLAN eraldi kaabliga ruuterisse | Kunagi (raiskab porte) |
| **Router-on-a-Stick** | Üks trunk link, subinterface'id | Väike võrk, pole L3 switchi |
| **L3 Switch (SVI)** | Switch teeb ise routingut | Suur võrk, kiirus oluline |

---

## Router-on-a-Stick

### Idee

Ruuteril on **üks füüsiline port**, aga mitu **loogilist subinterface'i**.

```
         Trunk (802.1Q)
              │
    ┌─────────┴─────────┐
    │      Router       │
    │  G0/0 (no IP!)    │
    │    │              │
    │  G0/0.10 ← VLAN 10 → 192.168.10.1/24
    │  G0/0.20 ← VLAN 20 → 192.168.20.1/24
    │  G0/0.30 ← VLAN 30 → 192.168.30.1/24
    └───────────────────┘
```

### Kuidas töötab?

1. Switch saadab trunk linki kaudu **tagged** frame (802.1Q)
2. Router loeb tag'i → teab mis VLAN
3. Router marsruudib → lisab uue tag'i
4. Switch saab frame'i → saadab õigesse VLANi

### Seadistamine - Router

```
Router(config)# interface G0/0
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# interface G0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.10.1 255.255.255.0
Router(config-subif)# exit

Router(config)# interface G0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.20.1 255.255.255.0
Router(config-subif)# exit

Router(config)# interface G0/0.30
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 192.168.30.1 255.255.255.0
```

**Tähtis:**
- `G0/0` - füüsiline port, **pole IP-d**, ainult `no shutdown`
- `G0/0.10` - subinterface, `.10` on lihtsalt number (soovitav = VLAN ID)
- `encapsulation dot1Q 10` - seob subinterface'i VLAN 10-ga
- IP-aadress = selle VLANi **gateway**

### Seadistamine - Switch

```
Switch(config)# interface G0/1
Switch(config-if)# switchport trunk encapsulation dot1q
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
```

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

---

## L3 Switch (SVI)

### Idee

L3 switch = switch + ruuter ühes seadmes.

**SVI (Switch Virtual Interface)** = virtuaalne interface iga VLANi jaoks.

```
    ┌─────────────────────────┐
    │      L3 Switch          │
    │                         │
    │  VLAN 10 ← SVI → 192.168.10.1/24
    │  VLAN 20 ← SVI → 192.168.20.1/24
    │  VLAN 30 ← SVI → 192.168.30.1/24
    │                         │
    │  Fa0/1 (VLAN 10)        │
    │  Fa0/2 (VLAN 20)        │
    │  Fa0/3 (VLAN 30)        │
    └─────────────────────────┘
```

### Eelised vs Router-on-a-Stick

| Router-on-a-Stick | L3 Switch |
|-------------------|-----------|
| Trunk link = bottleneck | Wire-speed routing |
| Eraldi seade | Kõik ühes |
| Odavam | Kallim |
| Väike võrk | Suur võrk |

### Seadistamine

```
Switch(config)# ip routing                    ! Lülita routing sisse

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

**Tähtis:**
- `ip routing` - ilma selleta switch ei marsruudi!
- `interface vlan 10` = SVI (mitte füüsiline port)
- VLAN peab olemas olema (`vlan 10` + pordid määratud)

### Kontrollimine

```
Switch# show ip route

C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
C    192.168.30.0/24 is directly connected, Vlan30
```

---

## Võrdlus

```
                    Router-on-a-Stick         L3 Switch (SVI)
                    
Topology:           [PC]──[Switch]══[Router]  [PC]──[L3 Switch]
                         trunk
                         
Bottleneck:         Trunk link (1Gbps)        Ei ole (wire-speed)

Config location:    Router subinterfaces      Switch SVIs

Käsk:               encapsulation dot1Q       ip routing + interface vlan

Hind:               Odavam                    Kallim

Kasutus:            SOHO, lab, väike võrk     Enterprise, campus
```

---

## Tõrkeotsing

### PC ei saa gateway'ni

1. `show vlan brief` - kas port õiges VLANis?
2. `show interfaces trunk` - kas trunk töötab?
3. `show ip interface brief` - kas SVI/subinterface on `up/up`?
4. Kas VLAN on trunk'il lubatud?

### VLANide vahel ei tööta

1. Kas routing on sees? (`ip routing` L3 switchil)
2. Kas PC gateway on õige?
3. `show ip route` - kas mõlemad võrgud näha?
4. `ping` gateway mõlemast VLANist

---

## Käskude kokkuvõte

### Router-on-a-Stick

| Käsk | Selgitus |
|------|----------|
| `interface G0/0.10` | Loo subinterface |
| `encapsulation dot1Q 10` | Seo VLAN 10-ga |
| `ip address 192.168.10.1 255.255.255.0` | Gateway IP |

### L3 Switch

| Käsk | Selgitus |
|------|----------|
| `ip routing` | Lülita routing sisse |
| `interface vlan 10` | Loo SVI |
| `ip address 192.168.10.1 255.255.255.0` | Gateway IP |
| `no shutdown` | Aktiveeri SVI |
