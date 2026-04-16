# DHCP ja DNS — dnsmasq seadistamine

## Kordamine

<details>
<summary>❓ Ei mäleta mis on DHCP? Loe siit.</summary>

DHCP jagab arvutitele IP-aadresse automaatselt. Server annab kliendile: IP-aadressi, maski, gateway ja DNS serveri.

Protsess: **DISCOVER** → **OFFER** → **REQUEST** → **ACK**.

**Kust lugeda:**
- Google: `dnsmasq dhcp-range syntax example`
- Google: `arch wiki dnsmasq` → osa "DHCP server"
- Cisco NetAcad ENSA moodul 8

</details>

<details>
<summary>❓ Ei mäleta mis on DNS? Loe siit.</summary>

DNS tõlgib nimesid IP-aadressideks. A-kirje seob nime ja IP: `server.labor.ee → 172.16.0.10`.

**Kust lugeda:**
- Google: `dnsmasq address record static DNS example`
- Google: `arch wiki dnsmasq` → osa "DNS"

</details>

<details>
<summary>❓ Ei mäleta kuidas ip helper-address töötab? Vaata siit.</summary>

DHCP broadcast ei läbi ruuterit. `ip helper-address` ütleb ruuterile: edasta broadcast unicastina serverile.

**Kust lugeda:**
- [Cisco: DHCP Relay Agent Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/xe-16/dhcp-xe-16-book/dhcp-relay-agent-xe.html) → osa "Configuring the DHCP Relay Agent" — seal on `ip helper-address` näide koos topoloogiaga
- IOS-is: `R1(config-if)# ip helper-address ?` — küsimärk näitab mida käsk ootab

</details>

### Kuidas dokumentatsioonist otsida?

| Sul on vaja | Kirjuta Google'isse |
|------------|---------------------|
| dnsmasq DHCP seadistus | `dnsmasq dhcp-range syntax example` |
| dnsmasq DHCP gateway ja DNS option | `dnsmasq dhcp-option 3 6 example` |
| dnsmasq DNS kirje | `dnsmasq address record static DNS example` |
| Cisco DHCP relay | `cisco ios ip helper-address configuration example` |
| dnsmasq praktiline juhend | `arch wiki dnsmasq` |
| vi kasutamine | `vi editor cheat sheet` |

---

## Mis on teisiti?

Sa tead juba mis on DHCP ja DNS. Seekord on kaks erinevust:

1. **DHCP server on eraldi masin** — dnsmasq Linuxi konteineris, mitte ruuteris endas
2. **Server on teises alamvõrgus** — seega vajad DHCP relay'd (`ip helper-address`), et broadcast jõuaks kohale

```
Klient (IP puudu)           R1                    dnsmasq (172.16.0.10)
   10.10.X.0/24         e0/0 | e0/1              172.16.0.0/24
                              |
        DHCP broadcast ──→  relay ──→  unicast ──→ dnsmasq vastab
```

---

## 1. DHCP relay ruuteril (Cisco IOS)

Mõtle: kust tuleb broadcast? Klientide poolt, ehk `e0/0`. Sinna pannaksegi `ip helper-address`.

```
R1(config)# interface e0/0
R1(config-if)# ip helper-address 172.16.0.10
```

See ütleb R1-le: "kui saad DHCP broadcasti sellel liidesel, saada see edasi aadressile `172.16.0.10`."

Tähtis: helper-address läheb **klientide poolsele** liidesele, mitte serveri poolsele.

> Cisco dokumentatsioon: otsi "ip helper-address" — [cisco.com](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/xe-16/dhcp-xe-16-book/dhcp-relay.html)

---

## 2. dnsmasq DHCP konfiguratsioon

Konfiguratsioonifail: `/etc/dnsmasq.d/lab.conf`

Mõtle: mida klient vajab?

| Klient vajab | dnsmasq seadistus | Mida sinna kirjutad |
|-------------|-------------------|---------------------|
| IP-aadress + mask | `dhcp-range=` | Kliendivõrgu vahemik, mitte serverivõrgu! |
| Default gateway | `dhcp-option=3,` | R1 klientide poolne liides |
| DNS server | `dhcp-option=6,` | dnsmasq enda IP |

### dhcp-range

```
dhcp-range=10.10.5.100,10.10.5.200,255.255.255.0,12h
            ──────────  ──────────  ─────────────  ───
            esimene IP   viimane IP   alamvõrgumask  kui kaua lease kehtib
            mida jagada  mida jagada
```

Iga osa komaga eraldatud. Vahemik peab olema **sinu kliendivõrgust**, mitte serverivõrgust. Gateway aadressi (`.1`) ära vahemikku pane.

### dhcp-option

```
dhcp-option=3,10.10.5.1
             ─  ─────────
             │  IP-aadress
             └─ option number: 3 = default gateway
```

```
dhcp-option=6,172.16.0.10
             ─  ──────────
             │  IP-aadress
             └─ option number: 6 = DNS server
```

Option numbrid tulevad DHCP standardist. Laboris läheb vaja ainult `3` ja `6`.

> Google: `dnsmasq dhcp-option 3 6 example`

---

## 3. DNS konfiguratsioon

DNS seob nime IP-aadressiga. dnsmasq-is:

```
address=/server.labor.ee/172.16.0.10
         ───────────────  ──────────
         domeeninimi       IP-aadress mida tagastatakse
```

Iga kirje eraldi rida. Kaldkriipsud eraldavad nime ja IP.

---

## 4. Reload

Pärast konfiguratsioonifaili muutmist ütle dnsmasq-ile "loe uuesti":

```
kill -HUP $(pidof dnsmasq)
```

Teenus ei peatu, lihtsalt loeb seaded uuesti sisse.

---

## 5. Ei oska vi-d?

Kui jääd `vi`-ga hätta, otsi **konkreetset probleemi**: `how to save and exit vi`, `how to edit file in vi`, `how to delete line in vi`. Mitte "vi cheat sheet" — see annab liiga palju infot.

---

## 6. Levinumad vead

| Viga | Sümptom | Lahendus |
|------|---------|----------|
| Vahemik vales võrgus | Klient saab vale IP | `dhcp-range` peab olema **kliendivõrgu** aadressid |
| Gateway puudu | Klient ei pääse teise võrku | Lisa `dhcp-option=3,...` |
| DNS puudu | `show ip` ei näita DNS-i | Lisa `dhcp-option=6,...` |
| helper-address vale liidesel | DHCP ei jõua serverini | Pane `e0/0`-le (klientide pool), mitte `e0/1`-le |
| helper-address vale IP | DHCP ei jõua serverini | IP peab olema **dnsmasq aadress**, mitte gateway |
| Reload unustatud | Muudatused ei rakendu | `kill -HUP $(pidof dnsmasq)` |
| `no shutdown` puudu | Liides on `administratively down` | IOL-XE pordid on vaikimisi väljas |
| Trükiviga konfis | dnsmasq ignoreerib rida | `cat /etc/dnsmasq.d/lab.conf` — vaata üle |
