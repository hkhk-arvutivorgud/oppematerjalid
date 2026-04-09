# GNS3 Labori Juhend - Tudengile

## Sisselogimine

1. Ava brauser (Chrome/Firefox)
2. Mine aadressile:
   - **Klassis:** `http://192.168.5.110:3080`
   - **Kaugtöö (VPN):** `http://192.168.100.110:3080`
3. Kasutajanimi: **sinu eesnimi** (nt `alar`, `alex`, `marii`)
4. Parool: `av2lab2025`

## Projekti avamine

Sul on juba projekt loodud! Pärast sisselogimist näed oma projekti (nt `lab1-alar`). Kliki sellel.

Uue projekti loomiseks: **+ New project** → anna nimi → **Create**

## Seadmete lisamine

1. Kliki vasakul **Browse all devices** (ülemine ikoon — kast ruudustikuga)
2. Näed seadmete nimekirja
3. **Lohista** seade töölauale (drag & drop)

| Seade | Kasutus | RAM |
|-------|---------|-----|
| **IOL XE L2** | Switch (VLAN, STP, trunk) | 256 MB |
| **IOL XE L3** | L3 Switch / Router (routing, DHCP, NAT) | 512 MB |
| **VPCS** | Lihtne PC (ping, IP seadistus) | ~0 |

## Seadme ümbernimetamine

1. Topeltkliki seadme **nimel** (nt "IOU1")
2. Kirjuta uus nimi (nt `SW1-ALAR`)
3. Vajuta Enter

⚠️ **Hostname peab sisaldama sinu nime!** See tõestab et sina tegid töö.

```
enable
configure terminal
hostname SW1-ALAR
```

## Kaablite ühendamine

1. Kliki vasakul **kaabli ikoonil** (joon kahe punkti vahel)
2. Kliki **esimesel seadmel** → vali port (nt `Ethernet0/0`)
3. Kliki **teisel seadmel** → vali port (nt `Ethernet0/0`)
4. Kaabel tekib!
5. Vajuta **ESC** kui oled valmis

### Kaabli kustutamine

Paremkliki kablil → **Delete**

### Pordid

**IOL XE L2/L3:**
- `e0/0` kuni `e0/3` — Ethernet pordid (grupp 0)
- `e1/0` kuni `e1/3` — Ethernet pordid (grupp 1)

**VPCS:**
- `eth0` — ainus port

## Seadmete käivitamine

1. Ülemine menüüriba → roheline **▶ Start** nupp (käivitab kõik seadmed)
2. Oota **~30-60 sekundit** kuni seadmed bootivad
3. Roheline täpp = töötab

## Konsooli avamine

1. **Paremkliki** seadmel
2. Vali **Web console** (avaneb uues aknas)
3. Kui näed `Switch>` või `Router>` — seade on valmis!
4. Kui ekraan on tühi — vajuta **Enter**

## Tüüpiline topoloogia

```
         L3SW (IOL XE L3)
         e0/0       e0/1
     (trunk)       (trunk)
         |            |
        SW1          SW2
     (IOL XE L2)  (IOL XE L2)
      e0/1 e0/2    e0/1 e0/2
       |     |      |     |
      PC1   PC2    PC3   PC4
     (VPCS)        (VPCS)
```

## Konfigureerimise näited

### VLAN loomine (switchis)

```
enable
configure terminal
vlan 10
 name Kontor
exit
vlan 20
 name Tootmine
exit
```

### Access port (PC ühendus)

```
interface e0/1
 switchport mode access
 switchport access vlan 10
exit
```

### Trunk port (switch ↔ switch)

```
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit
```

### SVI — gateway VLANile (L3 switchis)

```
interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
exit
ip routing
```

### VPCS käsud

```
ip dhcp                              ! Küsi DHCP-lt aadress
ip 10.10.10.10 255.255.255.0 10.10.10.1  ! Staatiline IP
show ip                              ! Näita IP infot
ping 10.10.10.1                      ! Ping
trace 10.10.20.10                    ! Traceroute
```

## Salvestamine

Projekt salvestub **automaatselt** serverisse (Ctrl+S ka töötab).

Seadme konfiguratsioon — salvesta eraldi igas seadmes:

```
copy running-config startup-config
```

## Projekti peatamine

1. Kliki punasel **■ Stop all** nupul
2. Või sulge brauser — projekt jääb serverisse alles
3. Järgmine kord: logi sisse → ava projekt → Start all → jätka

## Kasulikud show käsud

| Käsk | Mida näitab |
|------|-------------|
| `show vlan brief` | VLANid ja pordid |
| `show interfaces trunk` | Trunk pordid ja lubatud VLANid |
| `show ip interface brief` | IP aadressid ja pordi staatus |
| `show ip route` | Marsruutimistabel |
| `show running-config` | Kogu konfiguratsioon |
| `show ip dhcp binding` | DHCP jagatud aadressid |
| `show ip dhcp pool` | DHCP poolide staatus |
| `show mac address-table` | MAC aadressite tabel |

## Tõrkeotsing

### Seade ei booti

- Oota ~60 sek (IOL bootib aeglaselt)
- Paremkliki → **Stop**, oota 5 sek, **Start**
- Kui ikka ei tööta → küsi õpetajalt

### Konsool ei avane

- Kontrolli kas popup blocker on väljas
- Proovi: paremkliki → **Web console** (mitte topeltkliki)
- Proovi teise brauseriga

### Ping ei tööta — kontrolli järjekorras

1. `show ip interface brief` — kas IP on seadistatud?
2. `show vlan brief` — kas port on õiges VLANis?
3. `show interfaces trunk` — kas trunk töötab?
4. `show ip route` — kas marsruut on olemas?
5. Kas `ip routing` on sees? (L3 switchil)
6. Kas PC-l on gateway? → `show ip` (VPCS)

### DHCP ei tööta

- `show ip dhcp pool` L3 switchil — kas pool on olemas?
- `show interfaces trunk` — kas trunk lubab VLANi?
- VPCS: proovi `ip dhcp` uuesti

### "Port is already used"

- See port on juba kaabliga ühendatud
- Kustuta vana kaabel enne (paremkliki kablil → Delete)

## Reeglid

1. **Ära muuda** teiste projekte
2. **Hostname** peab sisaldama sinu nime
3. **Salvesta** konfiguratsioon (`copy run start`)
4. **Peata seadmed** kui lahkud (Stop all)
5. Probleemide korral **küsi õpetajalt**