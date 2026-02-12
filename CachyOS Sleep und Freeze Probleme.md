---
title: CachyOS Sleep und Freeze Probleme
created: 2026-02-12
tags:
  - linux
  - cachyos
  - intel-gpu
  - troubleshooting
status: solved
---

# CachyOS Sleep und Freeze Probleme

## Problem

**System**: CachyOS, KDE Plasma, Intel Sandy Bridge GPU (i915)

1. System geht zu schnell in Sleep-Modus
2. System friert nach Aufwachen aus Sleep ein → Hard-Reset nötig

## Ursache

- **Sleep-Timeout**: KDE Energieverwaltung zu kurz eingestellt
- **Freeze**: Intel i915 GPU Power Management Bug (RC6/FBC) bei Sandy Bridge GPUs

## Lösung

### 1. Sleep-Timeout anpassen
**Systemeinstellungen → Energieverwaltung → Energiesparen**
- Ruhezustand auf "Nie" oder gewünschte Zeit setzen

### 2. GPU Freeze beheben
```bash
# Backup erstellen
sudo cp /etc/default/grub /etc/default/grub.backup

# Kernel-Parameter hinzufügen
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet i915.enable_rc6=0 i915.enable_fbc=0"/' /etc/default/grub

# GRUB neu generieren
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Neustart
sudo reboot
```

**Parameter**:
- `i915.enable_rc6=0` → Deaktiviert RC6 Power Saving (verhindert Freeze)
- `i915.enable_fbc=0` → Deaktiviert Framebuffer Compression

**Trade-off**: Leicht erhöhter Stromverbrauch (~2-5W) für Stabilität

## Verifizierung

```bash
# Parameter aktiv?
cat /proc/cmdline  # sollte i915.enable_rc6=0 i915.enable_fbc=0 enthalten

# Suspend/Resume Test
systemctl suspend  # System aufwecken und testen
```

**Erfolgreich wenn**: System wacht normal auf, keine Freezes

## Rollback (falls nötig)

```bash
sudo cp /etc/default/grub.backup /etc/default/grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

## Referenz

[Arch Wiki: Intel Graphics Power Management](https://wiki.archlinux.org/title/Intel_graphics#Disable_Frame_Buffer_Compression)
