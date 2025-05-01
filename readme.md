#!/bin/bash
# Use RTX 3090 (04:00.0 / 04:00.1) inside an LXC container

DEVICES=(
  "0000:04:00.0"
  "0000:04:00.1"
)

echo ">>> Releasing from vfio-pci..."
for DEV in "${DEVICES[@]}"; do
  echo "$DEV" > "/sys/bus/pci/devices/$DEV/driver/unbind" 2>/dev/null || true
done

echo ">>> Clearing driver overrides..."
for DEV in "${DEVICES[@]}"; do
  echo "" > "/sys/bus/pci/devices/$DEV/driver_override"
done

echo ">>> Loading NVIDIA drivers..."
modprobe nvidia nvidia_modeset nvidia_uvm nvidia_drm snd_hda_intel

echo ">>> Rebinding RTX 3090 to NVIDIA driver..."
for DEV in "${DEVICES[@]}"; do
  echo "$DEV" > /sys/bus/pci/drivers_probe
done

echo ">>> Fixing permissions..."
chmod 0666 /dev/nvidia*

echo ">>> RTX 3090 now ready for LXC container."
