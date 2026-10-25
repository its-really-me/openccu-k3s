# OpenCCU auf k3s installieren (x86_64, HmIP-RFUSB im DualCoPro-Modus)

Diese Anleitung beschreibt die Installation von [OpenCCU](https://github.com/OpenCCU/OpenCCU) als Helm-Release in einem
k3s-Single-Node-Cluster auf einem x86_64-Host. Als Funkmodul dient ein **HmIP-RFUSB**-Stick, der über den
**raw-uart**-Treiber angebunden wird und damit Homematic IP **und** Homematic Classic (BidCoS) gleichzeitig bedient
(DualCoPro).

Getestet mit: Ubuntu 24.04, Kernel `6.8.0-138-generic`, k3s `v1.34.6+k3s1`, Helm-Chart `openccu/openccu` 3.89.11,
OpenCCU `3.89.11.20260919`, HmIP-RFUSB Firmware 4.4.18.

---

## 1. Voraussetzungen

- x86_64-Host mit Ubuntu 24.04 (Kernel ≥ 6.8 – auf 5.15 bauen die Kernel-Module nicht sauber)
- HmIP-RFUSB (USB-ID `1b1f:c020`) am Host, möglichst über ein kurzes Verlängerungskabel/Hub vom Gehäuse abgesetzt
- **Kein** HM-CFG-USB/HM-CFG-USB-2 (`1b1f:c00f`) gleichzeitig am Host – OpenCCU würde ihn sonst als BidCoS-Modul
  wählen und DualCoPro deaktivieren

---

## 2. Kernel-Module (piVCCU-DKMS)

OpenCCU braucht auf dem Host die Module `eq3_char_loop`, `generic_raw_uart` und `hb_rf_usb_2`.
Sie kommen aus dem piVCCU-Repository als DKMS-Paket.

```bash
sudo apt install wget ca-certificates build-essential bison flex libssl-dev gpg

wget -q -O - https://apt.pivccu.de/piVCCU/public.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/pivccu-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/pivccu-archive-keyring.gpg] https://apt.pivccu.de/piVCCU stable main" \
  | sudo tee /etc/apt/sources.list.d/pivccu.list

sudo apt update
sudo apt install linux-headers-$(uname -r) pivccu-modules-dkms
dkms status            # eq3_char_loop, generic_raw_uart, hb_rf_usb_2 → installed
```

Module beim Boot laden:

```bash
echo eq3_char_loop | sudo tee /etc/modules-load.d/eq3_char_loop.conf
printf "generic_raw_uart\nhb_rf_usb_2\n" | sudo tee /etc/modules-load.d/hb_rf_usb.conf
sudo reboot
```

Es wird **keine** udev-Regel benötigt: `1b1f:c020` ist per `modules.alias` bereits `hb_rf_usb_2` zugeordnet.
Den Stick nicht an `cp210x` binden.

**Prüfen:**

```bash
lsmod | grep -E "eq3_char_loop|generic_raw_uart|hb_rf_usb_2"
ls /sys/class/raw-uart/                    # → raw-uart
ls -la /dev/raw-uart*                      # → /dev/raw-uart (kein /dev/ttyUSB*)
sudo dmesg | grep -i raw-uart
# raw-uart raw-uart: Reset radio module
# raw-uart raw-uart: Registered new raw-uart device using underlying device eQ-3 HmIP-RFUSB@usb-…
```

---

## 3. k3s und Helm

```bash
curl -sfL https://get.k3s.io | sh -

# kubeconfig für den normalen Benutzer lesbar machen
echo 'write-kubeconfig-mode: "0644"' | sudo tee -a /etc/rancher/k3s/config.yaml
sudo systemctl restart k3s
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc && source ~/.bashrc

sudo snap install helm --classic
```

---

## 4. Namespace

OpenCCU muss privilegiert laufen (Zugriff auf Funkmodul und `eq3loop`):

```bash
kubectl create namespace homeautomation
kubectl label namespace homeautomation pod-security.kubernetes.io/enforce=privileged
```

---

## 5. Helm-Values

Die Datei liegt als [`openccu-values.yaml`](openccu-values.yaml) im Repository – nach `~/openccu-values.yaml`
auf den Host kopieren. Inhalt:

```yaml
image:
  repository: ghcr.io/openccu/openccu
  tag: 3.89.11.20260919            # feste Version statt latest

persistence:
  sys:                             # NICHT den Host-/sys mounten: sonst sieht der Pod die Host-Netzwerk-
    enabled: false                 # interfaces statt eth0 → kein IP-Status, Alarm "No Internet"
  data:
    enabled: true
    storageClass: "local-path"
    accessMode: ReadWriteOnce
    size: 2Gi

securityContext:
  privileged: true                 # Pod sieht damit /dev/raw-uart direkt, kein Device-Mount nötig

service:
  main:
    type: NodePort
    ports:
      http:
        enabled: true
        port: 80
        nodePort: 30180            # WebUI
      https:
        enabled: true
        port: 443
        nodePort: 30443
      rfd:                         # BidCoS-RF XML-RPC
        enabled: true
        port: 2001
        nodePort: 32001
      hmip:                        # HmIP-RF XML-RPC
        enabled: true
        port: 2010
        nodePort: 32010
      rega:                        # ReGa Skript-API
        enabled: true
        port: 8181
        nodePort: 30181
      virt-dev:                    # Virtuelle Geräte / Gruppen
        enabled: true
        port: 9292
        nodePort: 32092
```

Die NodePorts sind fest vergeben, damit Integrationen (ioBroker, Home Assistant …) stabile Ports haben.
Bei Bedarf an freie Ports im Bereich 30000–32767 anpassen.

---

## 6. Installation

```bash
helm repo add openccu https://openccu.github.io/OpenCCU
helm repo update

helm install openccu openccu/openccu --version 3.89.11 \
  -n homeautomation -f ~/openccu-values.yaml

kubectl rollout status deploy/openccu -n homeautomation
```

**Prüfen** – der Boot-Log muss das Funkmodul als raw-uart-Gerät für **beide** Protokolle zeigen:

```bash
kubectl logs -n homeautomation deploy/openccu | grep -iE "network|Identifying|multimacd|rfd|HMIPServer|ReGa|Finished"
```

Erwartet:

```
Starting network: eth0: link up, fixed, firewall, inet up, 10.42.x.x, OK
Identifying Homematic RF-Hardware: HmRF: HMIP-RFUSB/eQ-3 HmIP-RFUSB@usb-…, HmIP: HMIP-RFUSB/eQ-3 HmIP-RFUSB@usb-…, OK
Updating Homematic RF-Hardware: HMIP-RFUSB: 4.4.18, not necessary, OK
Starting multimacd: OK
Starting rfd: OK
Starting HMIPServer: OK
Starting ReGaHss: OK
Finished Boot: 3.89.11.20260919 (oci_amd64)
```

---

## 7. Einrichtung in der WebUI

1. WebUI öffnen: `http://<host>:30180/` und Ersteinrichtung durchführen (Passwort, Zeitzone usw.).
2. **Geräte anlernen:** *Geräte anlernen → HmIP Gerät anlernen → Anlernmodus*, dann am Gerät den Anlernmodus
   auslösen (beim HmIP-SWDO: Batterie einlegen, LED blinkt orange). Das Gerät erscheint im Posteingang.
3. **API-Zugriff für Integrationen:** Die OpenCCU-Firewall sperrt XML-RPC und Skript-API ab Werk.
   Unter *Einstellungen → Systemsteuerung → Firewall-Konfiguration* „Homematic XML-RPC API“ und
   „Remote Homematic-Script API“ auf „Eingeschränkt“ stellen und die erlaubten IP-Bereiche eintragen.
   Hinweis: Über den NodePort sieht OpenCCU als Quell-IP die Pod-Netz-Adresse `10.42.0.1`, nicht die echte
   Client-IP – daher `10.42.0.0/16` in die Liste aufnehmen (gilt dann für alle Clients, die über den NodePort kommen).
   Bei aktivierter Authentifizierung (*Systemsteuerung → Sicherheit*) brauchen Clients einen WebUI-Benutzer –
   am besten einen eigenen pro Integration anlegen.
4. **API von außen prüfen** (zsh; fragt das Passwort einmal ab):

   ```bash
   X='<?xml version="1.0"?><methodCall><methodName>listDevices</methodName><params></params></methodCall>'
   read -s "PW?Passwort: "; echo
   for p in 32010 32001; do
     printf "%s: " $p
     curl -s -m 10 -u "BENUTZER:$PW" -H Content-Type:text/xml -d "$X" -o /dev/null -w "HTTP %{http_code}, %{size_download} Bytes\n" http://<host>:$p/
   done
   curl -s -m 10 -u "BENUTZER:$PW" -d 'Write(dom.GetObject(ID_DEVICES).Count());' -w "  [HTTP %{http_code}]\n" http://<host>:30181/rega.exe
   unset PW
   ```

   Erwartet überall `HTTP 200`. `401` = Firewall offen, aber Zugangsdaten falsch; keine Verbindung = Firewall sperrt.

---

## 8. Backup

**CCU-Backup über die WebUI ist Pflicht.** Die Daten liegen auf einem `local-path`-PVC unter
`/var/lib/rancher/k3s/storage/` und sind bei einer Neuinstallation des Nodes verloren.
Das Backup enthält Geräte, Kopplungen und Schlüssel und lässt sich in jede OpenCCU einspielen.

Zusätzlich empfohlen: Host-Sicherung auf ein NAS per NFS.

```bash
sudo apt install nfs-common
showmount -e <NAS-IP>
sudo mkdir -p /mnt/nas-backup
echo "<NAS-IP>:/volume1/Backup  /mnt/nas-backup  nfs  defaults,vers=3,nolock,_netdev  0  0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a && df -h /mnt/nas-backup

BACKUP=/mnt/nas-backup/$(hostname)-backup-$(date +%Y%m%d)
sudo mkdir -p $BACKUP
sudo rsync -av /etc/rancher/                  $BACKUP/rancher/
sudo rsync -av /var/lib/rancher/k3s/storage/  $BACKUP/k3s-storage/
sudo rsync -av /var/lib/rancher/k3s/server/   $BACKUP/k3s-server/
sudo rsync -av /etc/modules-load.d/           $BACKUP/modules/
sudo rsync -av /home/$USER/                   $BACKUP/home/
```

`k3s-server/` enthält Cluster-Secrets – das Backup-Ziel entsprechend schützen.

---

## 9. Betrieb

**Update auf eine neue OpenCCU-Version**

```bash
helm repo update
helm search repo openccu/openccu            # neue Chart-Version / App-Version
# image.tag in ~/openccu-values.yaml auf die neue App-Version setzen, dann:
helm upgrade openccu openccu/openccu --version <chart-version> -n homeautomation -f ~/openccu-values.yaml
```

Vorher ein Backup erstellen (*Einstellungen → Systemsteuerung → Sicherheit → Backup erstellen*).

**Stick umgesteckt?** Danach immer den Pod neu starten – die laufenden Dienste halten sonst das alte Device:

```bash
kubectl rollout restart deploy/openccu -n homeautomation
```

**Nach einem Pod-Neustart** die WebUI neu öffnen (nicht neu laden) – alte Sessions sind ungültig und die
Oberfläche bleibt sonst bei „Konfiguration…“ hängen.

**Diagnose bei Funkproblemen** (temporär, danach zurücksetzen):

```bash
kubectl exec -n homeautomation deploy/openccu -- sh -c \
  'sed -i "s/^LOGLEVEL_HMIP=.*/LOGLEVEL_HMIP=DEBUG/; s/^LOGLEVEL_RFD=.*/LOGLEVEL_RFD=1/" /etc/config/syslog'
kubectl rollout restart deploy/openccu -n homeautomation
# HmIP:   /var/log/hmserver.log      Funk/BidCoS: /var/log/messages (multimac, rfd)
# Zurück: LOGLEVEL_HMIP=ERROR, LOGLEVEL_RFD=5
```

**Deinstallation**

```bash
helm uninstall openccu -n homeautomation
# Der PVC bleibt erhalten (Daten sicher) – nur löschen, wenn wirklich gewünscht:
kubectl delete pvc -n homeautomation --all
```

---

## 10. Bekannte Einschränkungen

| Thema | Details |
|---|---|
| `privileged: true` | Pflicht für den Zugriff auf Funkmodul und `eq3loop`; Namespace braucht das PSA-Label (Abschnitt 4) |
| Kernel | `eq3_char_loop` & Co. bauen erst ab Kernel 6.8 (Ubuntu 24.04) sauber |
| `local-path`-PVC | Daten an den Node gebunden – WebUI-Backups sind Pflicht (Abschnitt 8) |
| HM-CFG-USB(-2) | Nicht gleichzeitig am Host stecken – wird sonst statt des RFUSB als BidCoS-Modul gewählt |
| Quell-IP über NodePort | OpenCCU sieht `10.42.0.1` statt der Client-IP; Firewall-Freigabe gilt damit für alle NodePort-Clients |
| HmIP-HAP / DRAP | Bräuchten Layer-2-Zugriff (macvlan, z. B. Multus CNI) – nicht Teil dieser Anleitung |

---

## Referenzen

- [OpenCCU auf GitHub](https://github.com/OpenCCU/OpenCCU) · [Releases](https://github.com/OpenCCU/OpenCCU/releases)
- [Wiki: Installation Kubernetes](https://github.com/OpenCCU/OpenCCU/wiki/Installation-Kubernetes)
- [Helm-Chart auf Artifact Hub](https://artifacthub.io/packages/helm/openccu/openccu)
- [bjw-s common library](https://bjw-s-labs.github.io/helm-charts/docs/common-library/) (Basis des Charts)
- [piVCCU APT-Repository](https://apt.pivccu.de/piVCCU)
