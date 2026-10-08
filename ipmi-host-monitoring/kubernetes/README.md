# Prometheus-Anbindung

Der Standard ist ein `ServiceMonitor` für den IPMI-Exporter auf den Debian-Hosts.
Er verwendet den bereits vorhandenen, vom Prometheus Operator verwalteten
Kubelet-Service ausschließlich zur Ermittlung der Node-Adressen. Gescraped wird
`http://<Node-Adresse>:9290/metrics`, mit `job="ipmi-host"` und Node-Name als
`instance` und `node`. Es werden keine Exporter-Pods angelegt.

`ServiceMonitor.spec.selector` wählt Services aus. Die Auswahl der Nodes erfolgt
anschließend über die angehängten Node-Metadaten und die Relabeling-Regel.

Das bestehende Node-Label `feature.node.kubernetes.io/ipmi-address` entscheidet,
welche Nodes teilnehmen. Der Labelwert kann weiterhin die BMC-IP enthalten; diese
IP ist für den lokalen Host-Exporter kein Scrape-Ziel. Ein vorhandenes Label,
auch mit leerem Wert, genügt. Entferne es von Nodes ohne eingerichteten Hostdienst.

## Vor dem Anwenden

1. Hostdienst und `/metrics` auf jedem ausgewählten Node prüfen. Prometheus muss
   dessen Node-Adresse auf TCP 9290 erreichen. Der Exporter muss auf einer
   passenden Hostadresse lauschen; `127.0.0.1` ist aus Prometheus nicht erreichbar.
2. `../scripts/check-kubernetes.sh` ausführen. Der Standard erwartet einen
   Kubelet-Service in `kube-system` mit Label `k8s-app: kubelet` und Service-Port
   `https-metrics`. Passe `servicemonitor.yaml` an den vorhandenen Service an.
   Mehrere passende Services können doppelte Targets erzeugen.
   Prüfe auch die Node-Zuordnung: Bei Endpoints muss die Adresse einen
   `nodeName` oder einen `targetRef` auf die entsprechende Node haben; bei
   EndpointSlices prüfe `endpoints[].nodeName`. Ohne angehängte Node-Metadaten
   entfernt die Label-Regel das Target. Nutze dann die direkte Node-Discovery
   unten, statt Adressen statisch in dieses Repository einzutragen.
3. In `kustomization.yaml` den Namespace und die Beispiel-Labels an die tatsächlichen
   `Prometheus.spec.serviceMonitorNamespaceSelector` und
   `Prometheus.spec.serviceMonitorSelector` anpassen. `release: CHANGEME` ist ein
   Platzhalter, keine Vorgabe für euren Prometheus. Der Namespace muss existieren.
4. Die installierte ServiceMonitor-CRD muss `spec.attachMetadata.node` unterstützen;
   Prometheus muss mindestens Version 2.37.0 verwenden. Seine ServiceAccount
   benötigt außerdem Zugriff auf Services, Pods und die verwendeten
   Endpoints/EndpointSlices sowie `list/watch` auf Nodes. Bestehendes Kubelet-Monitoring
   stellt viele dieser Rechte normalerweise bereits bereit.

Das Prüfsystem zeigt `Prometheus.spec.version` und `spec.image`. Ist die Version
nicht gesetzt, prüfe die tatsächlich laufende Prometheus-Version bzw. das Image
des Prometheus-Pods. Ein erfolgreicher Diagnose-Aufruf allein bestätigt weder
Scrape-Erreichbarkeit noch die IPMI-Abfrage.

```sh
kubectl explain servicemonitor.spec.attachMetadata
kubectl kustomize kubernetes
kubectl apply --dry-run=server -k kubernetes
kubectl apply -k kubernetes
```

Die Befehle oben gelten vom Repository-Root aus. Die beiden letzten Befehle erst
nach Anpassung und Prüfung der Platzhalter ausführen. Das Standard-Kustomize
wendet ausschließlich den ServiceMonitor an; optionale RBAC und ScrapeConfig
werden nicht mit angewendet.

Der ServiceMonitor übernimmt die Discovery-Rolle des Prometheus-Objekts. Bei
`EndpointSlice` muss der Operator die Kubelet-EndpointSlices tatsächlich pflegen
und Prometheus sie lesen können. Eine Umstellung nur im ServiceMonitor erzeugt
keine EndpointSlices. Bearer-Token- und TLS-Einstellungen eines vorhandenen
Kubelet-ServiceMonitors werden hier nicht benötigt, weil der Host-Exporter über
HTTP angesprochen wird.

## Optional: direkte Node-Discovery

Wenn kein geeigneter Kubelet-Service existiert oder Node-Metadaten beim
ServiceMonitor nicht verfügbar sind, kann `optional/scrapeconfig.yaml`
stattdessen direkt Nodes entdecken. Dafür muss die ScrapeConfig-CRD installiert
sein und `spec.kubernetesSDConfigs` unterstützen. Prüfe die von eurem Cluster
angebotene API-Version (`v1alpha1` im Beispiel) mit `kubectl api-resources` und
`kubectl explain scrapeconfig.spec.kubernetesSDConfigs`.

Passe Namespace und Labels an `Prometheus.spec.scrapeConfigNamespaceSelector`
und `Prometheus.spec.scrapeConfigSelector` an. Die Node-Discovery verwendet die
von Prometheus ausgewählte Node-Adresse, vorrangig `InternalIP`, und ersetzt deren
Port durch 9290. Sie benötigt keinen Kubelet-Service. **Nur einen der beiden
Scrape-Jobs aktivieren.**

Fehlen ausschließlich Node-Leserechte, enthält `optional/nodes-rbac.yaml` eine
additive ClusterRole und ein Binding. Die beiden ServiceAccount-Platzhalter
müssen die tatsächliche Prometheus-ServiceAccount bezeichnen. Bestehende Rollen
nicht ersetzen und dieses Binding nicht vorsorglich anwenden.

## Prüfung in Prometheus

Unter **Status → Targets** den Job `ipmi-host` prüfen. `up{job="ipmi-host"}` zeigt
die HTTP-Erreichbarkeit. Die eigentliche Hardware-Abfrage wird mit
`ipmi_up{job="ipmi-host"}` geprüft: HTTP `up=1` kann trotz eines fehlgeschlagenen
IPMI-Collectors auftreten. Den alten DaemonSet-Scrape-Job bei der Umstellung
deaktivieren, damit die Hosts nicht doppelt gescraped werden.

## Primärquellen

- [Operator API: ServiceMonitor, AttachMetadata und Selector](https://prometheus-operator.dev/docs/api-reference/api/)
- [Operator: Migration des Kubelet-Monitorings auf EndpointSlices](https://prometheus-operator.dev/docs/platform/troubleshooting/)
- [Prometheus Kubernetes-Discovery und Node-Metadaten](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Operator: ScrapeConfig](https://prometheus-operator.dev/docs/developer/scrapeconfig/)
