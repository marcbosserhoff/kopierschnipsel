# Netzwerkregeln für den Host-Exporter

Der konfigurierte Scrape liest `/metrics`; die Hardware-Abfrage dafür verwendet
lokal `/dev/ipmi0`. Der Upstream-Exporter stellt jedoch weiterhin `/ipmi` bereit.
Ein fremder `target`-Parameter kann FreeIPMI zu einem LAN-Zugriff veranlassen,
obwohl das Modul `OPENIPMI` enthält. Eine lokale Modul-Konfiguration ist deshalb
keine technische Durchsetzung eurer LAN-Sperre.

Vor dem Rollout mit Server-Ops diese beiden Regeln sicherstellen:

1. **Eingang:** TCP 9290 auf den ausgewählten Node-Adressen nur für die im
   Host-Netz tatsächlich sichtbaren Prometheus-Quelladressen freigeben. CNI/SNAT
   berücksichtigen. Der Installer öffnet selbst keine Firewall-Ports.
2. **Ausgang:** IPMI-LAN/RMCP-Zugriff des Exporters auf die BMC-Netze sperren.
   Dafür vorhandene Netzwerk-ACLs oder passende Host-Firewall-/Dienstregeln nutzen.
   Ein gewöhnliches Kubernetes-`NetworkPolicy`-Objekt für einen Pod schützt
   diesen systemd-Hostdienst nicht automatisch.

Die konkreten Quell- und Zielnetze gehören in eure Infrastrukturkonfiguration.
Dieses Repository setzt keine vermuteten Firewall-Regeln auf dem Host und
ändert keine bestehenden Management-Zugriffsregeln.

## Optional: IP-Filter direkt für den systemd-Dienst

Wenn eure Hosts systemd-IP-Filter über cgroup/eBPF unterstützen, kann der Dienst
zusätzlich auf Loopback und die tatsächlichen Prometheus-Quellnetze begrenzt
werden. Ergänze dann im Abschnitt `[Service]` der vorhandenen Repository-Datei
`host/prometheus-ipmi-exporter.override.conf` zum Beispiel:

```ini
IPAddressDeny=any
IPAddressAllow=localhost
IPAddressAllow=192.0.2.64/26
```

`192.0.2.64/26` ist ein Dokumentationsnetz und muss ersetzt werden. Mehrere
`IPAddressAllow=`-Zeilen sind möglich. Die Regeln betreffen sowohl eingehende
als auch ausgehende IP-Pakete. BMC-Adressen dürfen nicht innerhalb der erlaubten
Netze liegen; sonst wären sie von diesem Filter ebenfalls zugelassen.

Diese Einstellungen in die **vorhandene** Drop-in-Datei im Repository einfügen
und mit dem Installer ausrollen. Ein weiteres fremdes Drop-in würde vom
Installer als Konfigurationskonflikt zurückgewiesen.

Bei fehlender Kernel-/eBPF-Unterstützung kann systemd den Filter nicht durchsetzen.
Deshalb Dienstjournal, tatsächliche Erreichbarkeit aus Prometheus und die
Egress-Sperre prüfen; die bloße Existenz der Direktiven ist kein Nachweis.
Vorhandene Netz-ACLs bleiben der verlässliche Rahmen für eure Management-Netze.

## Primärquellen

- [FreeIPMI: Auswahl des Inband-/Out-of-band-Zugriffs](https://github.com/chu11/freeipmi-mirror/blob/master/common/toolcommon/tool-common.c)
- [Exporter: Weitergabe des Zielhosts an FreeIPMI](https://github.com/prometheus-community/ipmi_exporter/blob/master/freeipmi/freeipmi.go)
- [systemd: IPAddressAllow und IPAddressDeny](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html)
