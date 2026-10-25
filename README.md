# OpenCCU auf k3s für Home Assistant

Dieses Repository beschreibt, wie [OpenCCU](https://github.com/OpenCCU/OpenCCU) – die freie Homematic-Zentrale –
als Container in einem **k3s-Single-Node-Cluster** betrieben wird, um Homematic- und Homematic-IP-Geräte in
**Home Assistant** einzubinden.

OpenCCU übernimmt Funk, Gerätekopplung und Verschlüsselung; Home Assistant greift über die XML-RPC-Schnittstellen
von OpenCCU auf die Geräte zu.

## Hardware

| Komponente | Details |
|---|---|
| Host | HP EliteDesk 800 G1 Desktop-PC |
| CPU | Intel Core i5-4570 (4 Kerne, x86_64) |
| RAM | 16 GB |
| Speicher | 240 GB SSD |
| Betriebssystem | Ubuntu 24.04 LTS, Kernel 6.8 |
| Cluster | k3s (Single Node) |

### Funkmodul

Benötigt wird ein **eQ-3 HmIP-RFUSB** (USB-ID `1b1f:c020`).

- Er wird über den **raw-uart**-Treiber (`hb_rf_usb_2` aus dem piVCCU-Projekt) angebunden.
- Im **DualCoPro-Modus** bedient er **Homematic IP und Homematic Classic (BidCoS)** gleichzeitig – ein Stick
  für beide Funkprotokolle.
- OpenCCU aktualisiert die Firmware des Sticks bei Bedarf automatisch (getestet mit 4.4.18).

Nicht geeignet als Zentralen-Funkmodul ist der **HM-CFG-USB / HM-CFG-USB-2** (`1b1f:c00f`) – das ist ein
reiner Konfigurationsadapter.

## Ergebnis

- OpenCCU 3.89.11 mit dem unveränderten Stock-Image aus `ghcr.io/openccu/openccu`
- WebUI unter `http://<host>:30180/`
- Feste Ports für Home Assistant und andere Integrationen:

  | Schnittstelle | Port |
  |---|---|
  | HmIP-RF (XML-RPC) | `32010` |
  | BidCoS-RF (XML-RPC) | `32001` |
  | Virtuelle Geräte / Gruppen | `32092` |
  | ReGa Skript-API | `30181` |
  | WebUI HTTPS | `30443` |

## Dateien

| Datei | Inhalt |
|---|---|
| [INSTALL.md](INSTALL.md) | Schritt-für-Schritt-Installation: Kernel-Module, k3s, Helm, Einrichtung, Backup, Betrieb |
| [openccu-values.yaml](openccu-values.yaml) | Helm-Values für das Chart `openccu/openccu` |

## Schnellstart

Die vollständige Anleitung steht in [INSTALL.md](INSTALL.md). Kurzfassung:

1. piVCCU-Kernel-Module (`pivccu-modules-dkms`) installieren und `eq3_char_loop`, `generic_raw_uart`,
   `hb_rf_usb_2` beim Boot laden
2. k3s und Helm installieren, Namespace mit PSA-Label `privileged` anlegen
3. `helm install openccu openccu/openccu -n homeautomation -f openccu-values.yaml`
4. In der WebUI Geräte anlernen, Firewall für die APIs freigeben und einen Benutzer für Home Assistant anlegen
5. In Home Assistant die Homematic-Integration mit Host und den Ports oben einrichten
