# Top-Board-Radxa-Raspberry-pi-5
fix: Escript modificado para correcto funcionamiento de raspberry pi 5 con radxa penta hat y Top Borad

# Rockpi-Penta Fix para Raspberry Pi 5

Fix para el control de ventilador del paquete `rockpi-penta` en Raspberry Pi 5 con Radxa TOP Board v1.2.

## Problema

El paquete oficial `rockpi-penta` v0.2.2 tiene dos bugs que causan que el ventilador permanezca encendido aunque la temperatura esté por debajo del umbral configurado:

1. **misc.py**: La función `fan_temp2dc()` retorna `0.999` (fan al 100%) cuando la temperatura no alcanza ningún nivel, en lugar de `0` (apagado).

2. **fan.py**: El thread de PWM por software (`tr()`) alterna el GPIO entre ON/OFF continuamente, incluso cuando el duty cycle es 0, manteniendo el fan parcialmente encendido.

## Solución

### misc.py

Línea 23 - Corregir orden de duty cycles:
```python
lv2dc = OrderedDict({'lv3': 1.0, 'lv2': 0.75, 'lv1': 0.5, 'lv0': 0.25})
```

Línea 155 - Retornar 0 cuando no se alcanza ningún nivel:
```python
return 0
```

### fan.py

Método `tr()` - Mantener GPIO apagado cuando duty=0:
```python
def tr(self):
    while True:
        if self.value[0] == 0:
            self.line.set_value(0)
            time.sleep(0.1)
        else:
            self.line.set_value(1)
            time.sleep(self.value[0])
            self.line.set_value(0)
            time.sleep(self.value[1])
```

Método `write()` - Setear value[0]=0 para señalizar apagado:
```python
def write(self, duty):
    if duty == 0:
        self.value[0] = 0
        self.value[1] = self.period_s
    else:
        self.value[1] = duty * self.period_s
        self.value[0] = self.period_s - self.value[1]
```

## Instalación

1. Instalar paquete oficial:
```bash
wget https://github.com/radxa/rockpi-penta/releases/download/v0.2.2/rockpi-penta-0.2.2.deb
sudo apt install -y ./rockpi-penta-0.2.2.deb
```

2. Aplicar fix:
```bash
sudo cp fan.py /usr/bin/rockpi-penta/fan.py
sudo cp misc.py /usr/bin/rockpi-penta/misc.py
sudo systemctl restart rockpi-penta.service
```

3. Configurar umbrales en `/etc/rockpi-penta.conf`:
```ini
[fan]
lv0 = 50
lv1 = 55
lv2 = 60
lv3 = 65
```

4. Reiniciar servicio:
```bash
sudo systemctl restart rockpi-penta.service
```

## Verificación

```bash
vcgencmd measure_temp
sudo systemctl status rockpi-penta.service
```

## Entorno probado

- Raspberry Pi 5
- Raspberry Pi OS Bookworm 64-bit
- Radxa TOP Board v1.2
- rockpi-penta v0.2.2

## Archivos

- `fan.py` - Script de control de ventilador corregido
- `misc.py` - Funciones auxiliares corregidas (opcional, incluir si modificaste)

## Créditos

Fix desarrollado por Eduardo Sepúlveda.
Paquete original: [radxa/rockpi-penta](https://github.com/radxa/rockpi-penta)
