#!/usr/bin/env bash
set -euo pipefail

# Crea una VM de Windows 10 en KVM/libvirt usando una ISO proporcionada por el usuario.
# Requisitos: qemu-kvm, libvirt, virt-install.

usage() {
  cat <<'USAGE'
Uso:
  ./crear_pc_virtual_windows10.sh --iso /ruta/Windows10.iso [opciones]

Opciones:
  --nombre NOMBRE         Nombre de la VM (default: RDP)
  --ram MB                RAM en MB (default: 8192)
  --cpu N                 Número de vCPU (default: 4)
  --disco-gb GB           Tamaño del disco virtual en GB (default: 80)
  --ruta-disco RUTA       Ruta del disco qcow2 (default: /var/lib/libvirt/images/<nombre>.qcow2)
  --rdp-usuario USER      Usuario que usarás en la app de Windows (default: Administrador)
  --rdp-contrasena PASS   Contraseña que usarás en la app de Windows (default: Cambiame123!)
  --rdp-nombre NOMBRE     Nombre para guardar en la app RDP (default: RDP)
  --mostrar-vnc           Muestra datos de conexión VNC al finalizar
  -h, --help              Mostrar ayuda

Ejemplo:
  ./crear_pc_virtual_windows10.sh --iso ~/Descargas/Win10.iso --nombre RDP --rdp-usuario Admin --rdp-contrasena 'ClaveSegura!2026'
USAGE
}

VM_NAME="RDP"
RAM_MB="8192"
VCPUS="4"
DISK_GB="80"
ISO_PATH=""
SHOW_VNC="false"
DISK_PATH=""
RDP_USER="Administrador"
RDP_PASSWORD="Cambiame123!"
RDP_PROFILE_NAME="RDP"

while [[ $# -gt 0 ]]; do
  case "$1" in
    --iso)
      ISO_PATH="${2:-}"
      shift 2
      ;;
    --nombre)
      VM_NAME="${2:-}"
      shift 2
      ;;
    --ram)
      RAM_MB="${2:-}"
      shift 2
      ;;
    --cpu)
      VCPUS="${2:-}"
      shift 2
      ;;
    --disco-gb)
      DISK_GB="${2:-}"
      shift 2
      ;;
    --ruta-disco)
      DISK_PATH="${2:-}"
      shift 2
      ;;
    --rdp-usuario)
      RDP_USER="${2:-}"
      shift 2
      ;;
    --rdp-contrasena)
      RDP_PASSWORD="${2:-}"
      shift 2
      ;;
    --rdp-nombre)
      RDP_PROFILE_NAME="${2:-}"
      shift 2
      ;;
    --mostrar-vnc)
      SHOW_VNC="true"
      shift
      ;;
    -h|--help)
      usage
      exit 0
      ;;
    *)
      echo "❌ Opción no reconocida: $1"
      usage
      exit 1
      ;;
  esac
done

if [[ -z "$ISO_PATH" ]]; then
  echo "❌ Debes indicar la ISO con --iso /ruta/archivo.iso"
  usage
  exit 1
fi

if [[ ! -f "$ISO_PATH" ]]; then
  echo "❌ La ISO no existe: $ISO_PATH"
  exit 1
fi

for bin in virt-install virsh qemu-img; do
  if ! command -v "$bin" >/dev/null 2>&1; then
    echo "❌ Falta dependencia: $bin"
    echo "   Instálala y vuelve a ejecutar el script."
    exit 1
  fi
done

if [[ -z "$DISK_PATH" ]]; then
  DISK_PATH="/var/lib/libvirt/images/${VM_NAME}.qcow2"
fi

if [[ -f "$DISK_PATH" ]]; then
  echo "❌ Ya existe el disco: $DISK_PATH"
  echo "   Usa --ruta-disco para otra ruta o elimina el disco existente."
  exit 1
fi

echo "➡️  Creando disco virtual (${DISK_GB}GB) en: $DISK_PATH"
sudo qemu-img create -f qcow2 "$DISK_PATH" "${DISK_GB}G" >/dev/null

echo "➡️  Creando VM '$VM_NAME'..."
# --os-variant win10 mejora defaults de hardware.
# --cdrom inicia instalador desde la ISO de Windows 10.
# --network default usa la red NAT por defecto de libvirt.
virt-install \
  --name "$VM_NAME" \
  --memory "$RAM_MB" \
  --vcpus "$VCPUS" \
  --cpu host-passthrough \
  --disk "path=$DISK_PATH,format=qcow2,bus=virtio" \
  --cdrom "$ISO_PATH" \
  --os-variant win10 \
  --network network=default,model=virtio \
  --graphics vnc,listen=0.0.0.0 \
  --video qxl \
  --sound ich9 \
  --boot cdrom,hd \
  --noautoconsole

echo "✅ VM creada. Continúa la instalación de Windows 10 desde la consola gráfica/VNC."
echo "   Nombre VM: $VM_NAME"
echo "   Disco VM : $DISK_PATH"

VM_IP_RAW="$(virsh domifaddr "$VM_NAME" --source lease 2>/dev/null | awk '/ipv4/ {print $4; exit}' || true)"
VM_IP="${VM_IP_RAW%%/*}"
if [[ -z "$VM_IP" ]]; then
  VM_IP="00.2030.303.0"
fi

echo
echo "📱 Datos para usar en Microsoft Remote Desktop (teléfono):"
echo "   Nombre (PC name): $RDP_PROFILE_NAME"
echo "   Usuario        : $RDP_USER"
echo "   Contraseña     : $RDP_PASSWORD"
echo "   IP / Host      : $VM_IP"
echo
echo "ℹ️  Importante: usuario/contraseña son los que debes configurar dentro de Windows 10."
echo "   Si la IP no sale todavía, termina la instalación y ejecuta:"
echo "   virsh domifaddr $VM_NAME --source lease"

if [[ "$SHOW_VNC" == "true" ]]; then
  VNC_INFO="$(virsh vncdisplay "$VM_NAME" 2>/dev/null || true)"
  if [[ -n "$VNC_INFO" ]]; then
    echo "   VNC display: $VNC_INFO"
    echo "   Ejemplo conexión local: localhost${VNC_INFO}"
  fi
fi
