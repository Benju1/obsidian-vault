---
title: CachyOS Sleep und Freeze Probleme beheben
created: 2026-02-12
updated: 2026-02-12
tags:
  - linux
  - cachyos
  - troubleshooting
  - hardware
  - intel-gpu
  - power-management
  - kde
aliases:
  - CachyOS einfrieren
  - Intel GPU Sleep Problem
  - Sandy Bridge Suspend Bug
status: solved
---

# CachyOS Sleep und Freeze Probleme

## Übersicht

Dokumentation zur Behebung von zwei kritischen Problemen auf einem CachyOS System mit Intel Sandy Bridge GPU (2. Generation):
1. System geht zu schnell in den Sleep-Modus
2. System friert nach dem Aufwachen aus dem Energiesparmodus ein

## Problembeschreibung

### Problem 1: Zu schneller Sleep-Modus
- Der PC wechselt zu schnell in den Energiesparmodus (Suspend)
- Unterbrechung der Arbeit durch ungewolltes Einschlafen des Systems

### Problem 2: System friert nach Sleep ein
- Nach dem Aufwachen aus dem Energiesparmodus reagiert das System nicht mehr
- Maus und Tastatur funktionieren nicht
- Bildschirm bleibt eingefroren
- Nur ein Hard-Reset (Resetknopf/Ausschalten) hilft

## Systemumgebung

- **OS**: CachyOS (Arch-basiert)
- **Kernel**: 6.12.68-1-cachyos-lts
- **Desktop**: KDE Plasma (Wayland)
- **GPU**: Intel HD Graphics (Sandy Bridge, 2. Generation)
  - Device ID: 0122
  - Treiber: i915
- **Hardware**: Desktop-PC mit Intel Core 2. Generation

## Ursachenanalyse

### Problem 1: Sleep-Timeout
- **Ursache**: KDE Plasma Energieverwaltung mit zu kurzen Standard-Timeouts
- **Komponente**: `powermanagementprofilesrc` Konfiguration

### Problem 2: Freeze nach Resume
- **Hauptursache**: Intel i915 GPU Power Management Bug bei Sandy Bridge GPUs
- **Technische Details**:
  - RC6 Power Saving State verursacht Instabilität beim Resume
  - Framebuffer Compression (FBC) kann zu Race Conditions führen
  - Bekanntes Problem bei Intel Sandy Bridge (2010-2012 Hardware)
- **Komponenten betroffen**:
  - i915 Kernel-Treiber
  - GPU Power Management States (RC6)
  - Framebuffer Compression

## Lösung

### Lösung 1: Sleep-Timeout anpassen

#### Über GUI (empfohlen):
```bash
# KDE Energieeinstellungen öffnen
kcmshell6 kcm_powerdevilprofilesconfig
```

**Manuelle Schritte**:
1. Systemeinstellungen öffnen
2. → Energieverwaltung
3. → Energiesparen
4. Folgende Werte anpassen:
   - **Bildschirm dimmen nach**: Nach Wunsch (z.B. 10 Minuten)
   - **Bildschirm ausschalten nach**: Nach Wunsch (z.B. 15 Minuten)
   - **Ruhezustand nach**: "Nie" oder sehr lange Zeit (z.B. 2 Stunden)

### Lösung 2: Intel GPU Freeze beheben

#### 1. GRUB-Konfiguration sichern
```bash
sudo cp /etc/default/grub /etc/default/grub.backup
```

#### 2. Kernel-Parameter hinzufügen
```bash
sudo nano /etc/default/grub
```

Zeile ändern von:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
```

zu:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet i915.enable_rc6=0 i915.enable_fbc=0"
```

**Oder per Kommandozeile**:
```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet i915.enable_rc6=0 i915.enable_fbc=0"/' /etc/default/grub
```

#### 3. GRUB neu generieren
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

#### 4. System neu starten
```bash
sudo reboot
```

## Kernel-Parameter Erklärung

| Parameter | Bedeutung | Effekt |
|-----------|-----------|--------|
| `i915.enable_rc6=0` | Deaktiviert RC6 GPU Power Saving | Verhindert instabile Power States beim Resume |
| `i915.enable_fbc=0` | Deaktiviert Framebuffer Compression | Vermeidet Race Conditions und Grafikfehler |

**Trade-offs**:
- ✅ **Vorteil**: Stabiles System, kein Einfrieren mehr
- ⚠️ **Nachteil**: Leicht erhöhter Stromverbrauch (auf Desktop minimal, ~2-5W)
- ⚠️ **Nachteil**: Minimal geringere GPU-Effizienz

## Verifizierung

### Nach dem Neustart prüfen:

#### 1. Kernel-Parameter aktiv?
```bash
cat /proc/cmdline
```

**Erwartete Ausgabe** (sollte enthalten):
```
... loglevel=3 quiet i915.enable_rc6=0 i915.enable_fbc=0 ...
```

#### 2. GPU-Treiber geladen?
```bash
lspci -k | grep -A 3 VGA
```

**Erwartete Ausgabe**:
```
00:02.0 VGA compatible controller: Intel Corporation 2nd Generation Core...
    Kernel driver in use: i915
```

#### 3. Suspend/Resume testen
```bash
# System in Suspend versetzen (oder über KDE Menü)
systemctl suspend
```

**Test-Checkliste**:
- [ ] System geht in Sleep-Modus
- [ ] Maus/Tastatur weckt System auf
- [ ] Bildschirm schaltet sich wieder ein
- [ ] Maus und Tastatur funktionieren normal
- [ ] Keine Einfrieren oder Hänger
- [ ] Anwendungen laufen weiter

#### 4. Logs auf Fehler prüfen (nach Suspend/Resume)
```bash
journalctl -b | grep -i "suspend\|resume\|freeze" | tail -20
```

**Gut**: Keine ERROR oder FAILED Meldungen

#### 5. GPU Power States prüfen (optional)
```bash
sudo cat /sys/kernel/debug/dri/0/i915_drpc_info
```

RC6 sollte disabled/0 sein.

## Troubleshooting

### Falls das Problem weiterhin besteht:

#### Alternative Kernel-Parameter testen:
```bash
# Noch konservativere Einstellungen
i915.enable_rc6=0 i915.enable_fbc=0 i915.enable_psr=0
```

#### Oder nur RC6 deaktivieren:
```bash
i915.enable_rc6=0
```

### Zurückrollen (falls nötig):
```bash
# GRUB Backup wiederherstellen
sudo cp /etc/default/grub.backup /etc/default/grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

## Weiterführende Informationen

### Betroffene Hardware
- Intel Core i3/i5/i7 2000-Serie (2010-2011)
- Intel HD Graphics 2000/3000
- Sandy Bridge Architektur

### Bekannte Bugs
- [Arch Wiki: Intel Graphics - Power Management](https://wiki.archlinux.org/title/Intel_graphics#Disable_Frame_Buffer_Compression)
- Kernel Bug Tracker: i915 RC6 Sandy Bridge Issues

### Verwandte Probleme
- [[GPU Troubleshooting]]
- [[Linux Power Management]]
- [[KDE Plasma Konfiguration]]

## Changelog

- **2026-02-12**: Initial documentation - Problem identifiziert und gelöst
  - i915 RC6 und FBC deaktiviert
  - KDE Energieverwaltung angepasst
  - System stabil nach mehreren Suspend/Resume Zyklen

---

**Status**: ✅ Gelöst
**Getestet**: 2026-02-12
**Hardware**: Intel Sandy Bridge (2. Gen)
**OS**: CachyOS + Kernel 6.12.68-1-cachyos-lts
