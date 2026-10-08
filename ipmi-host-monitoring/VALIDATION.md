# Prüfung des ausgelieferten Setups

Stand: 2026-10-08. Geprüft wurden die Dateien dieses Archivs; es wurde nichts
auf euren Hosts oder in eurem Cluster installiert.

Erfolgreich durchgeführt:

- Bash-Syntax für alle Shellskripte und die Hilfe der drei Betriebs-Skripte.
- ShellCheck 0.10.0 für alle Shellskripte.
- YAML-Parsing mit Prüfung auf doppelte Mapping-Schlüssel.
- Rendering der Standard-Kustomization mit Kustomize 5.6.0; Namespace und
  ServiceMonitor-Labels werden gesetzt, der Kubelet-Service-Selector bleibt erhalten.
- Prüfung der Port-Ersetzung für IPv4 und geklammerte IPv6-Adressen.
- Fachliche Prüfung der Paketpfade, lokalen FreeIPMI-Optionen und der
  Discovery-Voraussetzungen anhand der verlinkten Upstream-/Debian-Quellen.

Die systemd-Direktiven wurden fachlich geprüft. `systemd-analyze verify` kann in
dieser eingeschränkten Erstellungsumgebung wegen des fehlenden `/tmp` nicht laufen.
Der enthaltene CI-Validator führt diese Prüfung in einer isolierten Prüfeinheit
aus, wenn systemd-analyze und ein schreibbares `/tmp` vorhanden sind. Das prüft
Syntax und Unit-Einstellungen, nicht die tatsächliche Laufzeit-Isolation.

Vor dem Rollout auf euren Systemen ausführen:

1. Netzwerkfreigaben und BMC-Egress-Sperre mit Server-Ops sicherstellen.
2. `sudo ./scripts/install-host.sh` und `sudo ./scripts/check-host.sh` auf jedem
   ausgewählten Node; bei konkretem Listener die passende `--url` angeben.
3. Cluster-Voraussetzungen mit `./scripts/check-kubernetes.sh` prüfen, Platzhalter
   ersetzen und `kubectl apply --dry-run=server -k kubernetes` ausführen.
4. Nach Anwendung in Prometheus `up` und die einzelnen `ipmi_up`-Collectorwerte
   für den Job `ipmi-host` prüfen.

Hardwarezugriff, das effektive systemd-Hardening, die Firewall, eure installierten
CRD-Versionen und echte Prometheus-Targets konnten ohne Zugriff auf eure Umgebung
nicht geprüft werden.
