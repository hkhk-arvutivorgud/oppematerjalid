# Inter-VLAN marsruutimine

## Probleem

Eelmises peatükis õppisime, et VLAN eraldab võrgu loogilisteks osadeks. HR osakond ühes VLANis, Sales teises, IT kolmandas. See on hea, sest broadcast'id ei levi enam üle terve võrgu ja meil on parem kontroll selle üle, kes kellega suhelda saab.

Aga nüüd tekib probleem. Kujuta ette, et HR osakonna töötaja peab saatma faili Sales osakonna töötajale. HR on VLAN 10-s, Sales on VLAN 20-s. Mis juhtub kui HR arvuti proovib Sales arvutit pingida? Mitte midagi. Pakett ei lähe läbi.

Miks? Sest VLANid on erinevad broadcast domain'id. Teisisõnu need on täiesti erinevad võrgud. Ja mis seade ühendab erinevaid võrke? Ruuter. Ilma ruuterita ei saa erinevad VLANid omavahel rääkida. See on nagu kaks eraldi hoonet ilma ühenduseta. Inimesed mõlemas hoones eksisteerivad, aga nad ei saa üksteise juurde minna.

---

## Router-on-a-Stick

Üks lahendus on Router-on-a-Stick. Nimi kõlab naljakalt, aga kirjeldab hästi mida me teeme: ruuter istub "kepi otsas" ehk ühe ainsa lingi otsas.

![Router-on-a-Stick topoloogia](https://cdn.networklessons.com/wp-content/uploads/2015/07/router-on-a-stick-topology-two-hosts-two-vlans-one-router.png)

Ruuteri ja switchi vahel on üks trunk link. See trunk kannab kõiki VLANe. Ruuteri poolel loome subinterface'id, mis on virtuaalsed liidesed, üks iga VLANi jaoks. Füüsilisel pordil G0/0 pole IP-aadressi, aga G0/0.10 saab VLAN 10 gateway'ks ja G0/0.20 saab VLAN 20 gateway'ks.

Vaatame samm-sammult, mis juhtub kui VLAN 10 arvuti tahab saata paketti VLAN 20 arvutile. Esiteks näeb arvuti, et sihtkoht on teises võrgus, seega saadab paketi oma gateway'le. Switch saadab paketi trunk linki kaudu ruuterisse ja pakett on märgistatud VLAN 10 tag'iga. Ruuter võtab paketi vastu subinterface G0/0.10 kaudu, sest see kuulab VLAN 10 liiklust. Siis vaatab ruuter marsruutimistabelit ja näeb, et sihtvõrk on ühendatud läbi G0/0.20. Ruuter saadab paketi välja ja pakett saab nüüd VLAN 20 tag'i. Switch saab paketi trunk lingilt, näeb VLAN 20 tag'i ja saadab paketi õigesse access porti. See kõik toimub millisekunditega.

![Router-on-a-Stick subinterface'id](https://cdn.networklessons.com/wp-content/uploads/2015/07/router-on-a-stick-sub-interfaces-with-ip-addresses.png)

Seadistamine on lihtne. Switchi poolel teeme tavalise trunk pordi. Ruuteri poolel aktiveerime kõigepealt füüsilise pordi käsuga "no shutdown", aga me ei anna sellele IP-aadressi. Siis loome subinterface'id. Iga subinterface'i juures ütleme käsuga "encapsulation dot1Q" mis VLANiga see seotud on, ja anname IP-aadressi mis saab selle VLANi gateway'ks.

Router-on-a-Stick töötab hästi väikestes võrkudes. Aga tal on üks suur puudus: kõik VLANidevaheline liiklus käib läbi ühe trunk lingi. Kujuta ette kontorit kus on sada inimest ja nad kõik saadavad üksteisele faile. Kõik need paketid peavad minema läbi ühe lingi ruuterisse ja tagasi. See link muutub pudelikaelaks. Väikeses võrgus kuni paarkümmend inimest on Router-on-a-Stick suurepärane, aga suures ettevõttes on vaja midagi võimsamat.

---

## L3 Switch ja SVI

Suure võrgu lahendus on Layer 3 switch. See on switch mis oskab ka marsruutida, sisuliselt ruuter ja switch ühes karbis. Layer 3 switchil kasutame SVI-sid ehk Switch Virtual Interface. SVI on virtuaalne liides mis esindab tervet VLANi. Kui lood "interface vlan 10" ja annad sellele IP-aadressi, siis see IP-aadress saab VLAN 10 gateway'ks.

![Multilayer switch](https://cdn.networklessons.com/wp-content/uploads/2015/07/multilayer-switch-intervlan-routing-two-computers.png)

Erinevus Router-on-a-Stick'iga on see, et L3 switchil pole pudelikaela. Pole trunk linki mille kaudu kõik peaks käima. Marsruutimine toimub switchi sees, riistvara tasemel, väga kiiresti. Seda nimetatakse wire-speed routing'uks.

Seadistamine on isegi lihtsam kui Router-on-a-Stick. Kõigepealt lülitame routing funktsiooni sisse käsuga "ip routing". See käsk on kriitiline, ilma selleta switch lihtsalt ei marsruudi isegi kui SVI-del on IP-aadressid. Siis loome SVI-d käsuga "interface vlan 10" ja anname IP-aadressi. Oluline on ka "no shutdown" sest vaikimisi on SVI-d maas.

Päris elus näed mõlemat lahendust. Väiksemas filiaalikontoris võib olla Router-on-a-Stick sest see on odavam. Peakontoris ja andmekeskuses on kindlasti L3 switchid sest seal on palju liiklust ja kiirus on oluline.

---

## Tõrkeotsing

Kui VLANidevaheline suhtlus ei tööta, kontrolli järgmisi asju.

Kõigepealt kontrolli kas arvuti saab oma gateway'ni. Kui PC VLAN 10-s ei saa pingida oma gateway't 192.168.10.1, siis probleem on juba VLANi sees, mitte VLANide vahel. Kontrolli kas port on õiges VLANis ja kas link on üleval.

Teiseks kontrolli kas trunk töötab. Käsk "show interfaces trunk" näitab kas trunk on üleval ja millised VLANid on lubatud. Kui VLAN 20 pole trunk'il lubatud, siis VLAN 20 liiklus ei jõua ruuterisse.

Kolmandaks kontrolli kas subinterface'd või SVI-d on üleval. Käsk "show ip interface brief" näitab staatust. Kui subinterface on down, kontrolli kas füüsiline port on "no shutdown" olekus ja kas VLAN on trunk'il lubatud.

L3 switchil kontrolli kindlasti kas "ip routing" on sees. See unub tihti ära. Käsk "show ip route" näitab marsruutimistabelit ja kui see on tühi, siis routing pole sisse lülitatud.

Viimaseks kontrolli kas PC gateway on õige. Kui PC IP on 192.168.10.10 aga gateway on seatud 192.168.20.1, siis midagi ei tööta.

---

## Lisalugemine

- [NetworkLessons: Router-on-a-Stick](https://networklessons.com/switching/router-on-a-stick)
- [NetworkLessons: Multilayer Switch Inter-VLAN Routing](https://networklessons.com/switching/multilayer-switch-inter-vlan-routing)
