# IPMI-Monitoring als Debian-Hostdienst

Dieses Repository installiert den Debian-Exporter als systemd-Dienst auf jedem
IPMI-fähigen Kubernetes-Node. FreeIPMI liest die Hardware über `/dev/ipmi0`.
Prometheus im Cluster fragt `http://<Node-Adresse>:9290/metrics` ab. Der enthaltene
ServiceMonitor verwendet die vorhandenen Kubelet-Endpunkte zur automatischen
Ermittlung der Node-Adressen.

Das bestehende Label `feature.node.kubernetes.io/ipmi-address` wählt die Nodes aus.
Sein Wert wird **nicht** als Scrape-Adresse verwendet. BMC-Zugangsdaten sind für
diesen lokalen Zugriff nicht nötig. Der bisherige zentrale IPMI-Helm-Exporter
wird für diese Architektur nicht benötigt; es werden keine Exporter-Pods angelegt.

## Voraussetzungen

- Debian 12 oder 13 mit systemd; Installation auf dem Host mit `sudo`.
- Ein funktionsfähiges IPMI-Character-Device `/dev/ipmi0` auf den betreffenden Hosts.
- Prometheus Operator mit einer ServiceMonitor-CRD, die `spec.attachMetadata.node`
  unterstützt, und Prometheus ab Version 2.37.0.
- Ein vorhandener, vom Operator gepflegter Kubelet-Service samt passenden
  Endpoints/EndpointSlices. Standard im Beispiel: Namespace `kube-system`,
  Service-Label `k8s-app=kubelet`, Portname `https-metrics`.
- Prometheus darf die erforderlichen Kubernetes-Ressourcen und insbesondere
  Nodes lesen (`get`, `list`, `watch`).
- Prometheus erreicht die verwendeten Node-Adressen auf TCP 9290. Die Host-Firewall
  erlaubt diesen Port ausschließlich für die tatsächlich ankommenden
  Prometheus-Quelladressen; je nach CNI/SNAT können das Pod- oder Node-Adressen sein.
- Eure Netzwerkregeln sperren IPMI-LAN/RMCP-Verbindungen vom Exporter zu den BMCs.
  `OPENIPMI` allein deaktiviert den HTTP-Remote-Endpunkt des Exporters nicht;
  siehe [Netzwerkregeln](docs/network-policy.md).
- Für die Cluster-Prüfkommandos: aktuelles `kubectl`, `jq` und ein passender Kontext.

Wenn der passende Kubelet-Service fehlt, liegt unter `kubernetes/optional/` eine
alternative ScrapeConfig mit direkter Node-Discovery. Die Standardinstallation
verwendet ausschließlich den ServiceMonitor. Details stehen in
[kubernetes/README.md](kubernetes/README.md).

## 1. Konfiguration im Repository anpassen

| Datei | Anpassung |
| --- | --- |
| `kubernetes/kustomization.yaml` | Namespace des ServiceMonitors und Labels passend zu eurem Prometheus; `release: CHANGEME` ersetzen oder durch eure tatsächlichen Labels ersetzen. |
| `kubernetes/servicemonitor.yaml` | Namespace/Labels/Portname des Kubelet-Service prüfen; bei Bedarf anpassen. |
| `host/ipmi-host-exporter.default` | Listener; Standard `0.0.0.0:9290` für IPv4. Alternativ eine erreichbare Node-IP oder `[::]:9290` für IPv6 verwenden. |
| `host/ipmi_host.yml` | Collector-Auswahl und bei Bedarf hardwareabhängige Sensor-Ausnahmen. Standard: `bmc`, `ipmi`, `chassis`; keine pauschalen Sensor-Ausnahmen. |

Der ServiceMonitor erbt die Discovery-Rolle aus dem Prometheus-Objekt. Bei
`EndpointSlice` müssen die entsprechenden Kubelet-EndpointSlices vorhanden sein.
Ein bloßes Umschalten der Rolle erzeugt sie nicht. Bei IPv6-Targets muss auch der
Host-Listener IPv6 unterstützen. Das Port-Rewriting erhält vorhandene IPv6-Klammern.

## 2. Hostdienst installieren

Repository auf den betreffenden Debian-Node übertragen und im Repository-Verzeichnis
ausführen. Vorher die Erreichbarkeit von TCP 9290 auf Prometheus begrenzen.

```bash
sudo ./scripts/install-host.sh
sudo ./scripts/check-host.sh
```

Der Installer prüft Debian/systemd, das Character-Device, bestehende
Konfigurationskonflikte und den lokalen Sensorzugriff. Er installiert
`prometheus-ipmi-exporter`, `freeipmi-tools` und `curl` über die konfigurierten
apt-Repositories. Anschließend richtet er diese Dateien ein:

| Repository-Datei | Ziel auf dem Host |
| --- | --- |
| `host/ipmi_host.yml` | `/etc/prometheus/ipmi_host.yml` |
| `host/ipmi-host-exporter.default` | `/etc/default/ipmi-host-exporter` |
| `host/prometheus-ipmi-exporter.override.conf` | `/etc/systemd/system/prometheus-ipmi-exporter.service.d/override.conf` |

Verwendet wird der Paketdienst `prometheus-ipmi-exporter.service` mit dem Binary
`/usr/bin/prometheus-ipmi-exporter`. Der Drop-in setzt die lokalen Startparameter
ausdrücklich und ersetzt die Remote-Beispielkonfiguration des Debian-Pakets.
Eine vorübergehende Runtime-Maskierung verhindert beim erstmaligen Paketinstallieren
den Start mit den Paket-Defaults. Bei abgebrochener Erstinstallation hebt
der Installer seine Maskierung auf und deaktiviert den neu installierten Paketdienst,
damit die Remote-Beispielkonfiguration auch beim nächsten Boot nicht startet.

Einen bestimmten Listener kannst du auch beim Installieren setzen:

```bash
sudo ./scripts/install-host.sh --listen-address 192.0.2.11:9290
sudo ./scripts/check-host.sh --url http://192.0.2.11:9290/metrics
```

`192.0.2.11` ist eine Beispieladresse und muss durch die erreichbare Node-Adresse
ersetzt werden. Die Adresse im Label ist dafür nicht maßgeblich. Für reproduzierbare
Rollouts die Listener-Einstellung stattdessen im Repository bzw. in eurer
Host-Konfigurationsverwaltung pflegen.

Unveränderte Dateien werden beim erneuten Lauf übernommen. Abweichende vorhandene
Dateien werden ohne `--force` nicht überschrieben. Nach Prüfung deiner Änderungen:

```bash
sudo ./scripts/install-host.sh --force
```

Der Installer sichert ersetzte Dateien mit einem `.bak.<Zeitstempel>`-Suffix.
Bei Wiederholungen denselben Listener-Parameter verwenden, wenn er zuvor per CLI
gesetzt wurde. Fremde Units und zusätzliche Drop-ins werden nicht automatisch
gelöscht oder deaktiviert. `--force` ist nur für die drei verwalteten Dateien bzw.
die bewusste Übernahme des bestehenden Debian-Dienstes vorgesehen.

### Wenn `/dev/ipmi0` fehlt

Mit Server-Ops prüfen, welche IPMI-Treiber diese Hardware benötigt. Häufig sind
`ipmi_devintf` zusammen mit `ipmi_si` oder `ipmi_ssif` beteiligt. Dieses Repository
lädt keine Kernelmodule automatisch und ändert keine globale Modulkonfiguration.
Nach korrekter Host-Konfiguration muss folgender Aufruf funktionieren:

```bash
sudo ipmi-sensors -D OPENIPMI --driver-device=/dev/ipmi0
```

Wenn `ipmi-sensors` noch fehlt: `sudo apt install freeipmi-tools`.

## 3. Node-Auswahl prüfen

Die bereits vorhandenen Labels können unverändert weiterverwendet werden:

```bash
kubectl get nodes -l feature.node.kubernetes.io/ipmi-address \
  -L feature.node.kubernetes.io/ipmi-address
```

Nur Nodes mit installiertem und geprüftem Hostdienst sollten dieses Label tragen.
Der ServiceMonitor prüft die **Existenz** des Labels, unabhängig vom Wert. Auch
ein vorhandener leerer Wert gilt als Markierung.

Falls ihr weitere Nodes per NFD-Datei markieren wollt, enthält
`examples/nfd/ipmi-adresse` das Format:

```text
feature.node.kubernetes.io/ipmi-address=10.20.30.41
```

Die IP pro Host ersetzen und die Datei unter
`/etc/kubernetes/node-feature-discovery/features.d/ipmi-adresse` ablegen. Die
NFD-Quelle `local` muss aktiviert sein und dieses Host-Verzeichnis im Worker
sichtbar sein. **Nur eine IP als Dateiinhalte genügt nicht.** Der Dateiname ist
frei wählbar. Nach dem nächsten NFD-Lauf das resultierende Node-Label prüfen.

## 4. ServiceMonitor anwenden

Vom Repository-Root auf einem Rechner mit Cluster-Zugriff:

```bash
./scripts/check-kubernetes.sh
kubectl kustomize kubernetes
kubectl apply --dry-run=server -k kubernetes
kubectl apply -k kubernetes
```

Vor dem Anwenden `CHANGEME` ersetzen und prüfen, dass euer Prometheus den
ServiceMonitor-Namespace und die Labels tatsächlich auswählt. Die Selektoren
stehen in `Prometheus.spec.serviceMonitorNamespaceSelector` und
`Prometheus.spec.serviceMonitorSelector`. Eine feste Helm-Release-Bezeichnung
wird nicht vorausgesetzt. Der Ziel-Namespace muss bereits existieren.

`port: https-metrics` ist hier nur der Portname für die Discovery des
Kubelet-Service. Die Relabel-Regel ersetzt den tatsächlichen Zielport durch
`9290`; das Abfrageprotokoll ist `http` und der Pfad `/metrics`. Es erfolgt keine
IPMI-Abfrage über den Kubelet. Der ServiceMonitor benötigt deshalb keine
Kubelet-Bearer-Token oder Kubelet-TLS-Einstellungen.

Die optionale RBAC-Datei unter `kubernetes/optional/` nur verwenden, wenn
Node-Leserechte fehlen. Dafür die tatsächliche Prometheus-ServiceAccount
eintragen; bestehende Rollen bleiben erhalten. Der Check unterstützt:

```bash
./scripts/check-kubernetes.sh \
  --prometheus-service-account monitoring/EURE-PROMETHEUS-SERVICEACCOUNT
```

Diese Prüfung benutzt Kubernetes-Impersonation. Ein Fehler kann auch bedeuten,
dass deinem eigenen Benutzer die Impersonation-Rechte fehlen.

## 5. Ergebnis und Migration prüfen

In Prometheus unter **Status → Targets** nach `ipmi-host` suchen. Erwartet wird
pro ausgewähltem Node und Adressfamilie ein Target auf dessen Host-Port 9290.
Bei dualen Adressfamilien darauf achten, dass euer Kubelet-Service nicht dieselbe
Hardware über beide Familien doppelt in diesen Job aufnimmt.

```promql
up{job="ipmi-host"}
ipmi_up{job="ipmi-host"}
ipmi_temperature_celsius{job="ipmi-host"}
```

`up=1` bestätigt die HTTP-Abfrage. `ipmi_up` ist eine Metrik je Collector und
bestätigt dessen Hardware-Abfrage. Es ist möglich, dass `up=1`, aber ein
`ipmi_up{collector="..."}=0` gemeldet wird. Ohne ausgewählte Nodes gibt es keine
Scrape-Targets und entsprechend keine neue `ipmi_up`-Zeitreihe. Vorhandene alte
Zeitreihen können noch über ihren historischen Zeitraum abgefragt werden.

Zusätzliche Host-Diagnose:

```bash
sudo systemctl status prometheus-ipmi-exporter.service
sudo systemctl cat prometheus-ipmi-exporter.service
sudo journalctl -u prometheus-ipmi-exporter.service -n 100 --no-pager
sudo systemd-analyze security prometheus-ipmi-exporter.service
```

Beim Wechsel den alten DaemonSet-Exporter auf demselben Host stoppen, bevor der
neue Dienst Port 9290 bindet. Einen vorhandenen manuell installierten
`ipmi-exporter.service` ebenfalls gezielt deaktivieren. Die alten Scrape-Jobs
nach erfolgreicher Prüfung des neuen Jobs entfernen, um doppelte Messungen zu
vermeiden. Einen zentralen Helm-Exporter erst entfernen, wenn er keine weiteren
verwendeten Jobs mehr bedient. Das Repository führt diese Cluster-Löschungen
nicht aus, da deren Release- und Ressourcennamen installationsabhängig sind.

## Sicherheitsmodell

Der Hostdienst verwendet UID 0, weil FreeIPMI für Inband-Zugriffe normalerweise
den Root-Status prüft. Ein anderer Service-User allein löst dieses Problem nicht.
Das Setup leert CapabilityBoundingSet und AmbientCapabilities, setzt
NoNewPrivileges, beschränkt die Gerätezugriffe auf `/dev/ipmi0` und schützt das
Dateisystem. Der SDR-Cache liegt im von systemd angelegten Verzeichnis
`/var/cache/ipmi-exporter`.

Das IPMI-Device ermöglicht grundsätzlich auch Management-Kommandos. Diese
Konfiguration bietet keine hardwareseitige Beschränkung auf reine Sensor-Lesezugriffe.
Auch mit den systemd-Beschränkungen bleibt es ein Hostdienst mit UID 0.
Die HTTP-Schnittstelle des Exporters, einschließlich des upstream vorhandenen
`/ipmi`-Endpunkts, darf nur für Prometheus erreichbar sein. Ein Aufruf von
`/ipmi?target=<fremder-host>` kann **trotz `driver: OPENIPMI`** eine LAN-Abfrage
auslösen, weil FreeIPMI einen fremden Zielhost als Out-of-band-Ziel behandelt.
Die hier konfigurierte Prometheus-Abfrage verwendet ausschließlich `/metrics`.
Die Vorgabe gegen BMC-LAN-Zugriffe muss daher zusätzlich mit Netzwerkregeln
durchgesetzt werden. Die Installation ändert keine BMC-Netzwerkeinstellungen und
keine bestehenden Netzwerkzugriffsregeln; passende Regeln sind eine Voraussetzung.

## Änderungen prüfen und in GitLab einchecken

Die enthaltene CI prüft Bash-Syntax, ShellCheck, YAML einschließlich doppelter
Schlüssel und die systemd-Direktiven in einer isolierten Prüfeinheit. Sie deployt
nichts und braucht keine Cluster-Credentials. Für einen
lokalen Lauf unter Debian:

```bash
sudo apt install python3-yaml shellcheck
make validate
```

Anschließend die Dateien als Repository-Root einchecken, einschließlich
`.gitlab-ci.yml`, `.gitignore` und `.editorconfig`. Beispiel für ein neues,
leeres GitLab-Projekt; die Remote-URL ersetzen:

```bash
git init -b main
git add .
git commit -m "Add Debian host IPMI monitoring"
git remote add origin git@gitlab.example.com:infra/ipmi-host-monitoring.git
git push -u origin main
```

Paketupdates erfolgen über eure reguläre Debian-Updateverwaltung. Dieses
Repository pinnt keine Exporter-Version und lädt keine Upstream-Binaries herunter.
Die Dateien wurden statisch geprüft; die echte Hardware-Abfrage, das Host-Hardening
und die Cluster-CRD-Kompatibilität müssen beim beschriebenen Rollout geprüft werden.

## Primärquellen

- [Debian 12: prometheus-ipmi-exporter](https://packages.debian.org/bookworm/prometheus-ipmi-exporter)
- [Debian 13: prometheus-ipmi-exporter](https://packages.debian.org/trixie/prometheus-ipmi-exporter)
- [Debian-Paketdateien](https://packages.debian.org/bookworm/amd64/prometheus-ipmi-exporter/filelist)
- [IPMI exporter: lokale Konfiguration](https://github.com/prometheus-community/ipmi_exporter/blob/master/docs/configuration.md)
- [FreeIPMI: Inband-Root-Prüfung](https://github.com/chu11/freeipmi-mirror/blob/master/README.build)
- [FreeIPMI: Entscheidung zwischen Inband und LAN](https://github.com/chu11/freeipmi-mirror/blob/master/common/toolcommon/tool-common.c)
- [Prometheus Operator: API und ServiceMonitor](https://prometheus-operator.dev/docs/api-reference/api/)
- [Prometheus: Kubernetes-Discovery](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [NFD: lokale Feature-Dateien](https://kubernetes-sigs.github.io/node-feature-discovery/v0.19/usage/customization-guide.html)
- [systemd: Ausführungsbeschränkungen](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
- [systemd: Geräte- und Ressourcenrichtlinien](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html)
- [GitLab CI: YAML-Konfiguration](https://docs.gitlab.com/ci/yaml/)
