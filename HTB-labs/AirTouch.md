# AirTouch

Initieel lijkt het er op dat er niks op deze machine draait. Nmap ziet alleen
een SSH poort open. Mogelijk gaat het hier om een UDP poort ipv een TCP poort?

Met een UDP scan vind ik net wat meer:

```
PORT    STATE         SERVICE VERSION
68/udp  open|filtered dhcpc
161/udp open          snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
Service Info: Host: Consultant
```

Met `snmpwalk` kunnen we meer informatie vergaren:

```bash
snmpwal -v 1 \
    -c public \
    10.x.x.x

# Results in
# SNMPv2-MIB::sysDescr.0 = STRING: "The default consultant password is: RxBlZhLmOkacNWScmZ6D (change it after use it)"
# SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
# DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (219734) 0:36:37.34
# SNMPv2-MIB::sysContact.0 = STRING: admin@AirTouch.htb
# SNMPv2-MIB::sysName.0 = STRING: Consultant
# SNMPv2-MIB::sysLocation.0 = STRING: "Consultant pc"
# SNMPv2-MIB::sysORLastChange.0 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORID.1 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
# SNMPv2-MIB::sysORID.2 = OID: SNMP-MPD-MIB::snmpMPDCompliance
# SNMPv2-MIB::sysORID.3 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
# SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
# SNMPv2-MIB::sysORID.5 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
# SNMPv2-MIB::sysORID.6 = OID: TCP-MIB::tcpMIB
# SNMPv2-MIB::sysORID.7 = OID: IP-MIB::ip
# SNMPv2-MIB::sysORID.8 = OID: UDP-MIB::udpMIB
# SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
# SNMPv2-MIB::sysORID.10 = OID: NOTIFICATION-LOG-MIB::notificationLogMIB
# SNMPv2-MIB::sysORDescr.1 = STRING: The SNMP Management Architecture MIB.
# SNMPv2-MIB::sysORDescr.2 = STRING: The MIB for Message Processing and Dispatching.
# SNMPv2-MIB::sysORDescr.3 = STRING: The management information definitions for the SNMP User-based Security Model.
# SNMPv2-MIB::sysORDescr.4 = STRING: The MIB module for SNMPv2 entities
# SNMPv2-MIB::sysORDescr.5 = STRING: View-based Access Control Model for SNMP.
# SNMPv2-MIB::sysORDescr.6 = STRING: The MIB module for managing TCP implementations
# SNMPv2-MIB::sysORDescr.7 = STRING: The MIB module for managing IP and ICMP implementations
# SNMPv2-MIB::sysORDescr.8 = STRING: The MIB module for managing UDP implementations
# SNMPv2-MIB::sysORDescr.9 = STRING: The MIB modules for managing SNMP Notification, plus filtering.
# SNMPv2-MIB::sysORDescr.10 = STRING: The MIB module for logging SNMP Notifications.
# SNMPv2-MIB::sysORUpTime.1 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.2 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.3 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.4 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.5 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.6 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.7 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.8 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.9 = Timeticks: (0) 0:00:00.00
# SNMPv2-MIB::sysORUpTime.10 = Timeticks: (0) 0:00:00.00
# HOST-RESOURCES-MIB::hrSystemUptime.0 = Timeticks: (226244) 0:37:42.44
# End of MIB
```

Hier is vooral de `default consultant password` interessant. Met een kleine
check lijken we te kunnen inloggen via `ssh consultant@10.x.x.x` en dit
wachtwoord.

# WiFi

In de home folder van de consultant user vinden we een diagram voor een VLAN
setup. Kort samengevat hebben we:

- Een `consultant vlan`, dit is waar we op terechtkomen met SSH
- `tablets vlan` die met WiFi werkt, SSID=`AirTouch-Internet`
- `corp vlan` ook WiFi, SSID=`AirTouch-Office`

Met `airmon-ng start wlan0 && airodump-ng wlan0mon` haal ik de volgende data op:

```
 BSSID              PWR  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID

 6E:F7:E2:07:77:26  -28       22        0    0   9   54   WPA2 CCMP   PSK  MiFibra-24-D4VY
 BA:58:5E:38:A6:A4  -28       27        0    0   3   54        CCMP   PSK  MOVISTAR_FG68
 F0:9F:C2:A3:F1:A7  -28      191        6    0   6   54        CCMP   PSK  AirTouch-Internet
 A2:78:61:2C:7A:D4  -28      191        0    0   6   54        CCMP   PSK  WIFI-JOHN
 8E:F7:D1:19:A8:46  -28      455        0    0   1   54        TKIP   PSK  vodafoneFB6N

 BSSID              STATION            PWR   Rate    Lost    Frames  Notes  Probes

 (not associated)   28:6C:07:12:EE:A1  -29    0 - 1      0        2         AirTouch-Office
 (not associated)   C8:8A:9A:6F:F9:D2  -29    0 - 1      0        3         AccessLink,AirTouch-Office
 (not associated)   28:6C:07:12:EE:F3  -29    0 - 1      0        8         AirTouch-Office
 F0:9F:C2:A3:F1:A7  28:6C:07:FE:A3:22  -29    1 - 1      0        6
```
