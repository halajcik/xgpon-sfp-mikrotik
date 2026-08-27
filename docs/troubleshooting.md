# Troubleshooting

Věci, které při zprovoznění PRX126 + MikroTik + CETIN VLAN 848 reálně spadly. Pořadí podle toho, jak často to vypadá „stick je rozbitý“, a přitom je to switch.

## PPPoE: `terminating… peer is not responding`

Router posílá PADI na `OptikaVLAN848`, CRS je vidí na uplinku (`/interface bridge host print where vlan-id=848`), na sticku v `tcpdump` na `eth0_0` nic, GEM upstream nějaké pakety má, downstream nula.

**Příčina:** port se stickem není tagged člen VLAN 848. Často proto, že 848 je ve společném `bridge vlan` řádku s 10,20,30,… a ten řádek tagged seznam ONU port nemá. Untagged PVID (třeba 10) na tom portu 848 nepropustí.

**Fix:** VLAN 848 jako vlastní záznam, tagged včetně ONU portu.

```rsc
/interface bridge vlan add bridge=bridge vlan-ids=848 \
    tagged=bridge,<uplink-na-router>,<port-se-stickem>
```

Ověření: `pontop` GEM počítadla — po PADI/PADO musí růst TX i RX. Na CRS `current-tagged` u 848 obsahuje ONU port.

Hardware offload na PRX126 umí sežrat `tcpdump` na `eth0_0` (0 packetů i když provoz teče). Věř GEM čítačům a PPPoE monitoru, ne tcpdumpu.

## Po přesunu managementu na VLAN 30 ONU nejde pingnout

Port má `pvid=30`, CRS má `192.168.11.2/24`, stick `192.168.11.1/24`, ping z CRS i tak timeout.

**Příčina:** stejný port je ve statickém `bridge vlan` pro VLAN 30 uvedený jako **tagged**. Ingress ze sticku je untagged → PVID 30, to projde. Egress ze switche na stick je tagged 30 → LCT (`eth0_0_1_lct`) tag zahodí.

**Fix:** z tagged seznamu VLAN 30 ONU port vyndat. Untagged členství nech na dynamickém řádku z PVID.

```rsc
/interface bridge port print detail where interface~"xgpon"
# pvid=30, frame-types=admit-all

/interface bridge vlan print detail where vlan-ids~"30"
# tagged: NESMÍ obsahovat ONU port
# untagged: ONU port (klidně jen dynamic from pvid)
```

## LAN broadcast teče do PON

`tcpdump` na sticku ukazuje untagged ARP z `10.0.0.0/22`. To OLT nechce vidět.

**Příčina:** ONU port má PVID 10 (nebo je untagged člen VLAN 10). Stick mostí untagged dál do GEM.

**Fix:** PVID 30, management `192.168.11.0/24` jen na CRS. Router tuhle síť neroutuje.

## iPhone: `ERR_CONNECTION_TIMED_OUT`, notebook na WiFi OK

Ping ze sticku na iPhone (`10.0.3.x`) přitom chodí.

**Příčina není firewall na routeru**, když je `10.0.0.0/22` v LAN. iOS Private Relay / Limit IP Address Tracking bere `192.168.11.1` jako nielokální cíl a pokus jde mimo LAN → timeout. Chrome na iOS k tomu ještě nemá Local Network oprávnění a SYN vůbec neodešle.

HTTPS redirect na self-signed certifikát to ještě zhorší (`uhttpd.main.redirect_https=1`).

**Fix:**

1. `redirect_https=0` na sticku.
2. DNAT na CRS: `10.0.0.100:8111` → `192.168.11.1:80`.
3. Masquerade out `vlan_xgpon_management`, ať se SYN-ACK vrací na switch, ne na WiFi MAC, kterou CRS nezná.
4. V telefonu otevřít `http://10.0.0.100:8111/` včetně `http://`, ideálně Safari / anonymní okno.

VLAN 30 na routeru (connected `192.168.11.3/24`) ten iPhone problém neřeší, pokud cíl zůstane `192.168.11.1`. Navíc vznikne asymetrický návrat přes CRS.

## Asymetrický routing, když má VLAN 30 i router

Forward: telefon → router → CRS → ONU. Return: ONU default gw `192.168.11.2` → CRS pošle přímo do VLAN 10.

Notebook občas projde (CRS má jeho ARP ze SSH na `10.0.0.100`). Telefon ne — v ARP tabulce CRS není.

**Fix:** VLAN 30 jen na CRS. Router bez routy na `192.168.11.0/24`. Přístup jen NAT.

## Stick má hodiny o rok vedle, NTP nejede

`8311_ntp_servers` bez DNS, nebo DNS na CRS (`192.168.11.2`), který DNS neservíruje.

```bash
fw_setenv 8311_dns_server 10.0.0.1
fw_setenv 8311_ntp_servers 10.0.0.1
uci set system.ntp.enabled='1'
uci add_list system.ntp.server='10.0.0.1'
uci commit system
/etc/init.d/sysntpd restart
```

## `tcpdump` / `omci_pipe` / `sfp_i2c` na sticku visí

Dropbear na 8311 nemá pořádný exec kanál ani SFTP. Používej interaktivní shell, soubory tahaj přes `base64`. `8311-support.sh` umí zabalit diagnostiku do tar.gz.

## Optika

`pontop` / SFP diagnostika:

- **Rx power** — co stick vidí z OLT (přes splitter). Typicky řádově −15 až −25 dBm.
- **Tx power** — co stick sám vysílá směrem k OLT, ne výkon protistrany.

FEC a GTC errory mají zůstat na nule. Stav PLOAM mimo O5 znamená, že identita (SN/heslo/MIB) nesedí, ne že je špatně VLAN na switchi.

## Rychlý checklist

| Symptom | Kam se dívat |
| --- | --- |
| PPPoE peer not responding | CRS `bridge vlan` 848 tagged na ONU portu |
| 192.168.11.1 z CRS timeout | ONU port tagged ve VLAN 30 |
| ARP z 10.0.0.0/22 na sticku | PVID ONU portu |
| iPhone timeout na 192.168.11.1 | NAT `10.0.0.100:8111` + vypnout HTTPS redirect |
| GEM TX > 0, RX = 0 | OLT / VLAN 848 nechodí dolů |
| PLOAM ≠ O5 | `8311_gpon_sn` a zbytek identity |
