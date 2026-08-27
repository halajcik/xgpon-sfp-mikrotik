# Jak na vlastní optický SFP od CETIN (O2, T-Mobile, Vodafone)

Návod, jak nahradit ONT od operátora SFP+ stickem s čipem **PRX126** (WAS-110 / 8311 firmware) na **XGS-PON přípojce CETIN**. Platí pro všechny tři retail značky na téže síti: **O2, T-Mobile i Vodafone**. VLAN 848 je od CETIN, PPPoE údaje se liší podle operátora.

Stick **klonuje sériové číslo původního ONT**. CETIN OLT pouští zařízení podle GPON SN, které už na přípojce má — nové číslo sticku nikam nehlásíš a na technickou podporu s „vlastním ONT“ nemusíš. Původní krabičku po přepnutí odpoj, ať na vlákně nejsou dvě stejná SN najednou.

Dál: VLAN 848 (PPPoE) přes CRS do routeru, management sticku jen na switchi přes NAT.

Ověřeno na:

| Kus | Role |
| --- | --- |
| SFP+ ONU stick PRX126 | XGS-PON, 8311 community firmware |
| MikroTik CRS (RouterOS 7) | stick v SFP+, VLAN filtering, NAT na LuCI/SSH |
| MikroTik router (RouterOS 7) | PPPoE klient na VLAN 848, bez VLAN 30 |

Firmware: [8311-was-110-firmware-builder](https://github.com/djGrrr/8311-was-110-firmware-builder), instalace: [pon.wiki](https://pon.wiki/guides/install-the-8311-community-firmware-on-the-was-110/). Kdybys místo klonování chtěl hlásit **nové** SN sticku, je to jiná cesta — viz [Techforum — vlastní ONT na síti CETIN](https://www.techforum.cz/topic/64188-vlastn%C3%AD-ont-na-s%C3%ADti-cetin/).

## Topologie

```
                         XGS-PON (OLT CETIN)
                                |
                         SC/APC pigtail
                                |
                    +-----------+-----------+
                    |  CRS  sfp-…-xgpon     |
                    |  PVID 30  (untagged)  |
                    |  tagged VLAN 848      |
                    |                       |
                    |  192.168.11.2  VLAN30 |---- NAT ---- 10.0.0.100:8111  web
                    |  10.0.0.100    VLAN10 |              10.0.0.100:2211  ssh
                    +-----------+-----------+
                                | trunk
                                | tagged 10, 848, …
                    +-----------+-----------+
                    |  Router               |
                    |  vlan-lan 10.0.0.1    |
                    |  OptikaVLAN848        |
                    |  pppoe "optika"       |
                    |  VLAN 30 NENÍ         |
                    +-----------------------+
```

Stick má dvě logické cesty na stejném SFP:

1. **Datová** — tagged VLAN 848, GEM port od OLT, PPPoE končí až na routeru.
2. **Management (LCT)** — untagged Ethernet z ONU. Switch mu dá PVID 30, ať se LAN broadcasty z VLAN 10 necpou do PON.

Router VLAN 30 nevidí. Z LAN, WiFi i z iPhonu se na LuCI jde přes **`http://10.0.0.100:8111/`**.

## VLAN plán

| VLAN | Kde | K čemu |
| --- | --- | --- |
| 10 | LAN, untagged na přístupových portech | domácí síť `10.0.0.0/22` |
| 30 | jen CRS + stick | management ONU `192.168.11.0/24` |
| 848 | tagged: stick, CRS, router | CETIN PPPoE |

Čísla LAN/management si můžeš změnit. **848 je dané od CETIN.**

---

## 1. Stick (PRX126 / 8311)

### Klonování identity původního ONT

OLT stick pustí, když vysílá **stejné GPON serial number** jako ONT, které na přípojce doteď viselo. Firmware 8311 to umí nastavit (`8311_gpon_sn`). Tím se vyhneš objednávce „vlastní ONT“ u operátora a čekání, až CETIN zapíše nové SN.

Často je potřeba zkopírovat i **HW verzi, SW verzi a Equipment ID** — OLT je u některých profilů kontroluje. Vezmi je z původního ONT (štítek, web, `omci` dump), ne z krabice sticku.

Do `8311_gpon_sn` patří tvar **4 ASCII písmena + 8 hex číslic**, celkem 12 znaků, například `HWTC33AABBCC`. **Ne** 16znakový čistý hex.

```
fw_setenv 8311_gpon_sn HWTC33AABBCC
```

Po změně identity reboot. Registrace na OLT = stav **O5** (PLOAM Associated):

```
pontop -b
# PLOAM status: O5.1 / O5
```

### Převod hexadecimálního SN na tvar pro stick

Na štítku nebo ve webu ONT bývá sériové číslo jako **16 hex znaků** (8 bajtů). První 4 bajty jsou výrobce v ASCII, zbylé 4 bajty zůstanou hex.

Příklad:

```
hex z ONT:  48 57 54 43 33 AA BB CC
            -- -- -- -- ----------
            H  W  T  C  33AABBCC

do sticku:  HWTC33AABBCC
```

| Hex | Význam |
| --- | --- |
| `48 57 54 43` | ASCII → `H W T C` |
| `33 AABBCC` | necháš jako hex `33AABBCC` |

PowerShell:

```
$h = '4857544333AABBCC'
$vendor = -join @(0,2,4,6 | ForEach-Object { [char][Convert]::ToInt32($h.Substring($_,2), 16) })
$vendor + $h.Substring(8).ToUpper()
# HWTC33AABBCC
```

Python:

```
h = '4857544333AABBCC'
print(bytes.fromhex(h[:8]).decode('ascii') + h[8:].upper())
```

Když ONT už ukazuje `HWTC33AABBCC`, nic nepřevádíš. Když zadáš do sticku plný hex `4857544333AABBCC`, firmware to špatně rozparsuje a OLT uvidí jiné SN, než čeká.

### Management IP

LCT je untagged. IP drž v rozsahu, který **není** tvoje LAN — jinak se ti do PON začnou sypat ARP z domácí sítě.

```bash
fw_setenv 8311_ipaddr 192.168.11.1
fw_setenv 8311_netmask 255.255.255.0
fw_setenv 8311_gateway 192.168.11.2
fw_setenv 8311_dns_server 10.0.0.1
fw_setenv 8311_ntp_servers 10.0.0.1
fw_setenv 8311_timezone Europe/Prague
fw_setenv 8311_ping_ip 192.168.11.2
fw_setenv 8311_dying_gasp_en 1
fw_setenv 8311_https_redirect 0
fw_setenv 8311_fix_vlans 1
```

A rovnou, bez rebootu:

```
uci set network.lct.ipaddr='192.168.11.1'
uci set network.lct.netmask='255.255.255.0'
uci set network.lct.gateway='192.168.11.2'
uci set network.lct.dns='10.0.0.1'
uci commit network
ip route replace default via 192.168.11.2 dev eth0_0_1_lct

uci set system.ntp.enabled='1'
uci -q delete system.ntp.server
uci add_list system.ntp.server='10.0.0.1'
uci commit system
/etc/init.d/sysntpd enable
/etc/init.d/sysntpd restart
```

`dying_gasp_en=1` obsadí UART RX (GPIO 508). Sériová konzole pak nejde používat jako vstup — to je záměr, OLT dostane dying gasp při výpadku napájení.

### LuCI z mobilu

uhttpd defaultně přesměruje na HTTPS a self-signed certifikát. Na iPhonu to končí timeoutem nebo varováním, které nejde odklikat. Vypni redirect:

```bash
fw_setenv 8311_https_redirect 0
uci set uhttpd.main.redirect_https='0'
uci set uhttpd.main.rfc1918_filter='0'
uci commit uhttpd
/etc/init.d/uhttpd restart
```

I tak **neotvírej** `http://192.168.11.1` z iPhonu. iOS Private Relay / Limit IP Tracking bere cizí RFC1918 rozsah jako internet a požadavek zmizí. Používej NAT na switchi (níž).

---

## 2. CRS — VLAN filtering

Předpoklad: bridge má `vlan-filtering=yes`.

### Port se stickem

```rsc
/interface bridge port set [find interface=sfp-sfpplus2-xgpon] \
    pvid=30 frame-types=admit-all ingress-filtering=yes trusted=yes
```

- **PVID 30** — untagged snímky ze sticku = management.
- **admit-all** — musí projít i tagged 848.
- **Nenechávej PVID 10.** Jinak untagged LAN (ARP, mDNS, DHCP) poteče do ONU a dál do PON.

### VLAN 848 jako samostatný řádek

Tohle je nejčastější důvod, proč PPPoE visí v `terminating… peer is not responding`.

Špatně — jeden řádek `vlan-ids=10,20,30,40,50,60,99,848` a ONU port v něm **není** tagged:

```
tagged=bridge,uplink,…
# sfp-sfpplus2-xgpon chybí → 848 na stick nejde
```

Správně — 848 zvlášť a stick **je** tagged:

```rsc
/interface bridge vlan add bridge=bridge vlan-ids=848 \
    tagged=bridge,qsfpplus1-1-router,sfp-sfpplus2-xgpon
```

VLAN 30 na tom portu nechej na dynamickém záznamu z PVID (`untagged=sfp-sfpplus2-xgpon`). Do statického tagged seznamu VLAN 30 ten port **nedávej**: switch by na egress posílal tag 30 a LCT sticku (untagged) by na to nereagoval. Pak `192.168.11.1` nejde pingnout, i když PVID je 30.

### Management IP na CRS

```rsc
/interface vlan add name=vlan_xgpon_management vlan-id=30 interface=bridge
/ip address add address=192.168.11.2/24 interface=vlan_xgpon_management
```

Router tuhle VLAN nemá. Stick má default gw `192.168.11.2`.

---

## 3. NAT — web a SSH jen přes switch

iPhone na WiFi (`10.0.0.0/22`) se na `192.168.11.1` nedostane, notebook na stejné WiFi ano. ICMP na telefon přitom chodí. Jde o to, že `192.168.11.0/24` není lokální podsíť telefonu.

Řešení: DNAT na LAN adresu CRS.

```rsc
/ip firewall nat add chain=dstnat protocol=tcp dst-address=10.0.0.100 \
    dst-port=8111 action=dst-nat to-addresses=192.168.11.1 to-ports=80 \
    comment="ONU web"
/ip firewall nat add chain=dstnat protocol=tcp dst-address=10.0.0.100 \
    dst-port=2211 action=dst-nat to-addresses=192.168.11.1 to-ports=22 \
    comment="ONU ssh"
/ip firewall nat add chain=srcnat out-interface=vlan_xgpon_management \
    action=masquerade comment="ONU mgmt masq"
```

| Služba | Adresa |
| --- | --- |
| LuCI | `http://10.0.0.100:8111/` |
| SSH | `ssh -p 2211 root@10.0.0.100` |

Masquerade je nutný: odpověď ze sticku se musí vrátit na CRS (192.168.11.2), ne rovnou na WiFi klienta. CRS jinak nemá ARP na telefon a SYN-ACK se ztratí.

www na CRS nech vypnuté (`/ip service disable www`), ať port 80 na switchi nikdo neplete s 8111.

---

## 4. Router — jen PPPoE

Přípojka je vždy CETIN, internetová VLAN je vždy **848**. Liší se jen PPPoE jméno a heslo podle toho, kdo ti službu fakturuje:

| Operátor | Uživatel | Heslo |
| --- | --- | --- |
| O2 | `O2` | `O2` |
| Vodafone | `vf` | `vf` |
| T-Mobile | vlastní účet | vlastní heslo |

T-Mobile nemá univerzální `O2`/`vf`. Účet je typicky ve tvaru `optin……@optin.t-mobile.cz`; heslo máš ve smlouvě, nebo ho pošle podpora. (Oficiální tabulka T-Mobile pro FTTx přes CETIN převodník uvádí i `adsl` / `adsl` — pokud osobní účet nemáš, zkus tohle.)

```
/interface vlan add name=OptikaVLAN848 vlan-id=848 interface=bridge
/interface pppoe-client add name="optika internet" interface=OptikaVLAN848 \
    user="O2" password="O2" add-default-route=yes use-peer-dns=yes
```

U Vodafone dej `vf`/`vf`, u T-Mobile svůj účet. VLAN 848 musí být tagged na uplinku ke CRS. MTU 1492.

Na routeru:

- žádný `vlan-xgpon-mgmt`
- žádná routa `192.168.11.0/24` ani `192.168.11.1/32`
- VLAN 30 pryč z `bridge vlan` tagged seznamu

---

## 5. Kontrola

Na CRS:

```rsc
/ping 192.168.11.1 count=4
/interface bridge vlan print where vlan-ids=848
/interface bridge port print detail where interface~"xgpon"
```

U VLAN 848 musí být `current-tagged` včetně portu se stickem. U portu `pvid=30`.

Na sticku (přes `ssh -p 2211`):

```bash
ip route          # default via 192.168.11.2
pontop -b         # O5, GEM port má TX i RX
```

Na routeru:

```rsc
/interface pppoe-client monitor 0 once
# status: connected
```

Z notebooku i z telefonu:

```text
http://10.0.0.100:8111/
```

Přímé `http://192.168.11.1` z LAN má timeout — to je správně.

---

## Proč to takhle

Podrobnosti a slepé uličky jsou v [docs/troubleshooting.md](docs/troubleshooting.md). Krátce:

1. **PPPoE nejede** — 848 není tagged na portu se stickem.
2. **ONU nejde pingnout po přesunu na VLAN 30** — port je zároveň tagged člen VLAN 30.
3. **Broadcast z LAN v PON** — PVID 10 na ONU portu.
4. **iPhone timeout, notebook OK** — cizí RFC1918 + Private Relay; proto NAT na `10.0.0.100`.
5. **Asymetrický routing přes router** — nechej VLAN 30 jen na CRS.

## Licence

MIT. Doplň si vlastní GPON SN, hesla a názvy portů. Do gitu nepatří sériová čísla ONT ani password hashe.
