# Guía Completa de Instalación y Validación
## Gateway LoRaWAN WM1302 + Raspberry Pi 5 + TTS

---

# 1. Introducción

Este documento describe paso a paso el procedimiento completo para desplegar un Gateway LoRaWAN funcional utilizando:

- Raspberry Pi 5
- Raspberry Pi OS Bookworm Legacy 64-bit
- Concentrador LoRaWAN WM1302 de Seeed Studio
- Pi Hat oficial de Seeed Studio
- The Things Stack (TTS)
- Packet Forwarder SX1302 HAL

La guía incluye:

- Instalación del sistema operativo
- Habilitación de interfaces
- Compilación del HAL
- Corrección de incompatibilidades del reset GPIO para Raspberry Pi 5
- Configuración del packet forwarder
- Conexión exitosa con TTS
- Validación funcional completa

---

# 2. Arquitectura General

La arquitectura utilizada es:

```text
End Devices LoRaWAN
        │
        │ LoRa 915 MHz
        ▼
WM1302 + Raspberry Pi 5
(Packet Forwarder)
        │
        │ UDP 1700
        ▼
The Things Stack (TTS)
(Network Server)
```

---

# 3. Hardware Utilizado

## 3.1 Raspberry Pi

- Raspberry Pi 5
- Alimentación oficial recomendada
- Ethernet recomendado para estabilidad

## 3.2 Concentrador LoRaWAN

- Wio WM1302 Gateway Module
- US915
- SX1302 + SX1250
- SPI

## 3.3 Pi Hat

- WM1302 Pi HAT v1.1
- Seeed Studio

---

# 4. Sistema Operativo

## 4.1 Imagen utilizada

Sistema operativo utilizado:

```text
Raspberry Pi OS Lite Legacy (64-bit)
Debian Bookworm
```

IMPORTANTE:

NO utilizar:

- Debian Trixie
- Raspberry Pi OS Trixie
- Configuraciones desktop innecesarias

Se identificaron problemas de compatibilidad GPIO y reset con Trixie.

---

# 5. Instalación del Sistema Operativo

## 5.1 Descarga de Raspberry Pi Imager

Descargar:

https://www.raspberrypi.com/software/

---

## 5.2 Configuración de Raspberry Pi Imager

Seleccionar:

### Device

```text
Raspberry Pi 5
```

### Operating System

```text
Raspberry Pi OS Lite Legacy (64-bit)
```

### Storage

Seleccionar microSD o SSD.

---

## 5.3 Configuración avanzada

Presionar:

```text
CTRL + SHIFT + X
```

Configurar:

### Hostname

```text
gateway-lora
```

### SSH

```text
Enable SSH
Use password authentication
```

### Username

```text
admin
```

### Password

Definir contraseña.

---

# 6. Primer Arranque

## 6.1 Acceso SSH

```bash
ssh admin@gateway-lora.local
```

O:

```bash
ssh admin@IP_DEL_GATEWAY
```

---

# 7. Actualización del Sistema

```bash
sudo apt update
sudo apt upgrade -y
```

---

# 8. Habilitación de Interfaces

## 8.1 SPI

```bash
sudo raspi-config
```

Ir a:

```text
Interface Options
→ SPI
→ Enable
```

---

## 8.2 I2C

```bash
sudo raspi-config
```

Ir a:

```text
Interface Options
→ I2C
→ Enable
```

---

## 8.3 Reinicio

```bash
sudo reboot
```

---

# 9. Validación SPI

## 9.1 Verificación

```bash
ls /dev/spidev*
```

Resultado esperado:

```text
/dev/spidev0.0
/dev/spidev0.1
/dev/spidev10.0
```

---

# 10. Validación I2C

```bash
ls /dev/i2c*
```

Resultado esperado:

```text
/dev/i2c-1
/dev/i2c-13
/dev/i2c-14
```

---

# 11. Instalación de Dependencias

```bash
sudo apt install -y git build-essential
```

---

# 12. Descarga del SX1302 HAL

```bash
cd ~
git clone https://github.com/Lora-net/sx1302_hal.git
cd sx1302_hal
```

---

# 13. Compilación del HAL

```bash
make clean
make
```

---

# 14. Validación de Binarios

```bash
ls -l util_chip_id/chip_id
ls -l packet_forwarder/lora_pkt_fwd
```

---

# 15. Problema Crítico Detectado

## 15.1 Incompatibilidad Raspberry Pi 5

El archivo original:

```text
tools/reset_lgw.sh
```

NO funciona correctamente en Raspberry Pi 5.

Problemas detectados:

```text
/sys/class/gpio incompatible
GPIO mapping incorrecto
Falla SX1250_0 en STANDBY_RC
chip version 0x05
```

---

# 16. Solución Correcta GPIO Raspberry Pi 5

## 16.1 Mapping Correcto Detectado

El mapping correcto para el Pi Hat WM1302 v1.1 fue:

```bash
SX1302_RESET_PIN=17
SX1302_POWER_EN_PIN=18
SX1261_RESET_PIN=5
AD5338R_RESET_PIN=13
```

IMPORTANTE:

El mapping genérico Semtech:

```bash
23 / 22 / 18
```

NO funcionó correctamente.

---

# 17. Reemplazo de reset_lgw.sh

## 17.1 Backup

```bash
cd ~/sx1302_hal/tools
cp reset_lgw.sh reset_lgw.sh.bak
```

---

## 17.2 Nuevo reset_lgw.sh

Crear:

```bash
nano ~/sx1302_hal/tools/reset_lgw.sh
```

Contenido:

```bash
#!/bin/bash

SX1302_RESET_PIN=17
SX1302_POWER_EN_PIN=18
SX1261_RESET_PIN=5
AD5338R_RESET_PIN=13

set_high() {
    pinctrl set "$1" op dh
}

set_low() {
    pinctrl set "$1" op dl
}

case "$1" in
    start)
        echo "CoreCell power enable through GPIO${SX1302_POWER_EN_PIN}..."
        set_high "$SX1302_POWER_EN_PIN"
        sleep 0.2

        echo "CoreCell reset through GPIO${SX1302_RESET_PIN}..."
        set_high "$SX1302_RESET_PIN"
        sleep 0.2
        set_low "$SX1302_RESET_PIN"
        sleep 0.2

        echo "SX1261 reset through GPIO${SX1261_RESET_PIN}..."
        set_low "$SX1261_RESET_PIN"
        sleep 0.2
        set_high "$SX1261_RESET_PIN"
        sleep 0.2

        echo "CoreCell ADC reset through GPIO${AD5338R_RESET_PIN}..."
        set_low "$AD5338R_RESET_PIN"
        sleep 0.2
        set_high "$AD5338R_RESET_PIN"
        sleep 0.2
        ;;

    stop)
        echo "CoreCell power disable through GPIO${SX1302_POWER_EN_PIN}..."
        set_low "$SX1302_POWER_EN_PIN"
        ;;

    *)
        echo "Uso: $0 {start|stop}"
        exit 1
        ;;
esac

exit 0
```

---

## 17.3 Permisos

```bash
chmod +x ~/sx1302_hal/tools/reset_lgw.sh
```

---

# 18. Enlaces Simbólicos

## 18.1 util_chip_id

```bash
ln -sf ../tools/reset_lgw.sh ~/sx1302_hal/util_chip_id/reset_lgw.sh
```

## 18.2 packet_forwarder

```bash
ln -sf ../tools/reset_lgw.sh ~/sx1302_hal/packet_forwarder/reset_lgw.sh
```

---

# 19. Validación del Concentrador

## 19.1 Lectura del EUI

```bash
cd ~/sx1302_hal/util_chip_id
sudo ./chip_id
```

Resultado esperado:

```text
Note: chip version is 0x10 (v1.0)
INFO: concentrator EUI: 0x0016c001f114dbb7
```

---

# 20. Registro del Gateway en TTS

## 20.1 Parámetros utilizados

### Gateway ID

```text
wm1302-gateway
```

### Gateway EUI

```text
0016C001F114DBB7
```

### Frequency Plan

```text
US915 FSB2
```

### Gateway Server Address

```text
186.154.15.54
```

### Require Authentication

```text
Disabled
```

---

# 21. Configuración Packet Forwarder

## 21.1 Copiar configuración base

```bash
cd ~/sx1302_hal/packet_forwarder
cp global_conf.json.sx1250.US915.USB global_conf.json
```

---

## 21.2 Modificar comunicación SPI

Editar:

```bash
nano global_conf.json
```

Modificar:

```json
"com_type": "SPI",
"com_path": "/dev/spidev0.0"
```

---

## 21.3 Configuración Gateway

Modificar:

```json
"gateway_ID": "0016C001F114DBB7",
"server_address": "186.154.15.54",
"serv_port_up": 1700,
"serv_port_down": 1700
```

---

# 22. Ejecución Packet Forwarder

```bash
cd ~/sx1302_hal/packet_forwarder
sudo ./lora_pkt_fwd
```

---

# 23. Resultado Esperado

## 23.1 Comunicación Correcta

```text
INFO: [up] PUSH_ACK received
INFO: [down] PULL_ACK received
```

---

## 23.2 Estado del Gateway

```text
### SX1302 Status ###
```

Con contadores incrementando:

```text
SX1302 counter (INST)
```

---

## 23.3 Temperatura

```text
### Concentrator temperature: 36 C ###
```

---

# 24. Estado Final Alcanzado

El gateway quedó completamente operativo.

Validaciones exitosas:

```text
SPI operativo                     ✅
I2C operativo                     ✅
SX1302 operativo                  ✅
SX1250 operativo                  ✅
Packet Forwarder operativo        ✅
UDP 1700 operativo                ✅
Conexión con TTS                  ✅
PUSH_ACK correcto                 ✅
PULL_ACK correcto                 ✅
US915 operativo                   ✅
```

---

# 25. Problema Raíz Identificado

El problema principal fue:

```text
GPIO mapping incorrecto para Raspberry Pi 5
```

El mapping genérico del HAL Semtech NO es compatible directamente con el Pi Hat WM1302 v1.1 en Raspberry Pi 5.

---

# 26. Recomendaciones

## 26.1 Backup inmediato

Respaldar:

```bash
~/sx1302_hal/tools/reset_lgw.sh
~/sx1302_hal/packet_forwarder/global_conf.json
```

---

## 26.2 Usar Ethernet

Recomendado:

```text
Ethernet sobre WiFi
```

para estabilidad del gateway.

---

## 26.3 No usar Trixie

Evitar:

```text
Debian Trixie
Raspberry Pi OS Trixie
```

para este gateway.

---

# 27. Próximos Pasos

Ahora el sistema está listo para:

- conectar nodos LoRaWAN reales
- pruebas OTAA
- uplinks/downlinks
- integración con TTS
- integración con plataformas IoT superiores
- monitoreo RF
- pruebas de cobertura

---

# 28. Referencias

## Semtech SX1302 HAL

https://github.com/Lora-net/sx1302_hal

---

## Raspberry Pi OS

https://www.raspberrypi.com/software/

---

## The Things Stack

https://www.thethingsindustries.com/docs/

---

## Seeed Studio WM1302

https://wiki.seeedstudio.com/

