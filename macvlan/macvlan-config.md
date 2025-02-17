# How to SetUp macvlan
Hier folgt eine Beschreibung, wie man macvlan auf der Synology einrichtet. Das ganze als eine "Zusammanfassung" Die Beschreibung gilt für die Adress 192.168.178.20 sollten andere IP-Adressen benötigt werden so muss es dem jeweiligen Netzwerk angepasst werden.
Der DHCP Server ist so konfiguriert, dass er IP-Adressen erst abe dem Bereich 192.168.178.100 - 200 vergibt.

## 1 - IP -Adressbereich definieren

[IP-Calculator](https://jodies.de/ipcalc?host=192.168.178.16&mask1=28&mask2=)

[![image](https://github.com/user-attachments/assets/04a68ff9-8037-49ba-8b6b-83df5a5a8094)


- Im Router/DHCP-Server sicher stellen, dass der Adressbereich nicht anderweitig verwendet wird. 
## 1 - Schnittstelle herausfinden

Sich mittels SSH mit der Synology verbinden und den Befehl 'ifconfig" ausführen
```bash
ifconfig
```
- bei der Ausgabe prüfen an  welcher Schnittstelle **eth0** oder **ovs_eth** oder bound_xxx die IP-Adresse der Synology vorkommt.
## 2 - macvlan_conf anlegen

- Name: macvlan_conf
- Driver: macvlan
- Parent network Card: ova_eth0 / eth0
- Subnet: 192.168.2.178.0/24
- IP-range 192.168.178.16/28
- Gateway: 192.168.178.1

![image](https://github.com/JoeFri2k/Synology/blob/f6d8e8a6ed0899e9b4fcf0c9f9e0af251090e483/macvlan/img/macvlan_conf.png)

## 3 - macvlan Netzerstellen

Hier wählt man wieder neues Netz anlegen selektiert jedoch die Option **Creation**
![image](https://github.com/JoeFri2k/Synology/blob/f6d8e8a6ed0899e9b4fcf0c9f9e0af251090e483/macvlan/img/macvlan_network.png)

## 4 -  Container dem Netzwerk hinzufügen compose.yml (stackfile)

```yaml
# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # For DHCP it is recommended to remove these ports and instead add: network_mode: "host"
    hostname: pihole
#################################################################
    networks:
      mcvlan_ovs_eth0:
        ipv4_address: 192.168.178.20
#################################################################
    ports:
      - 53:53/tcp
      - 53:53/udp
#     - "67:67/udp" # Only required if you are using Pi-hole as your DHCP server
      - 80:80/tcp
    environment:
      TZ: 'Europe/Berlin'
      # WEBPASSWORD: 'set a secure password here or it will be random'
    # Volumes store your data between container upgrades
    volumes:
      - '/volume1/docker/pihole/etc:/etc/pihole'
      - '/volume1/docker/pihole/dnsmasq:/etc/dnsmasq.d'
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
#    cap_add:
#     - NET_ADMIN # Required if you are using Pi-hole as your DHCP server, else not needed
    restart: unless-stopped
#################################################################
networks:
  mcvlan_ovs_eth0: # Definition des Netzwerks
    external: true # Gibt an, dass das Netzwerk bereits existiert
#################################################################
```




## 5 - macvlan und Netzwerkkarte "bridgen"

die Befehle, kann man der Reihe nach ausführen und testen ob dies das funktioniert. Wenn das alles geklappt hat dann das ganze in einen Skript packen und dieses automatisch mit dem Start der Synology ausführen. 

Das "cryptische" in der While-Schleife ist eine Funktion die darauf wartet, dass der Dockerservice gestartet ist, und wenn das der Fall ist, dann wir das Skript zu ende ausgeführt.

```shell
#!/bin/bash
while ! /usr/local/bin/docker info >/dev/null 2>&1;
do
	sleep 5s
done

ip link add mvl-brg link ovs_eth0 type macvlan mode bridge
ip addr add 192.168.178.30/32 dev mvl-brg
ip link set mvl-brg up
ip route add 192.168.178.16/28 dev mvl-brg
```
Wenn alles erfolgreich abgelaufen ist, dann sollte man jetzt den Container unter seiner eigenen Adresse anpingen können. 





----







