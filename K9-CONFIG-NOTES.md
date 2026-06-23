# EasyThreed K9 — Notas de configuración (custom)

Documentación del firmware Marlin adaptado a esta K9 modificada (sin endstops
físicos, con sonda Z de nivelación). Generado durante la sesión de reconfiguración.

## Hardware

| Componente | Detalle |
|---|---|
| Placa | **MKS Robin Lite** (`BOARD_MKS_ROBIN_LITE`), STM32F103 |
| Entorno build | `mks_robin_lite_maple` → `mksLite.bin` |
| Drivers | **A4988** en X/Y/Z/E (NO TMC → sin sensorless, sin StallGuard) |
| Control de corriente | **PWM** sobre el Vref de los A4988 — `M907` SÍ funciona |
| Pines corriente PWM | XY=`PB0`, Z=`PA7`, E=`PA6` (XY comparten un canal) |
| Default boot current | `PWM_MOTOR_CURRENT {800,800,800}` (XY, Z, E) en mA aprox. |
| Endstops físicos | **NINGUNO** (se quitaron; M119 da `x/y/z_min: open`) |
| Sonda Z | Reciclada de impresora de papel, **brazo basculante** sobre eje X, **deploy manual**. En pin `Z+` (Z_MAX). VERIFICADO: sin contacto=LOW=open, contacto=HIGH=triggered → `Z_MIN_PROBE_ENDSTOP_INVERTING=false` |
| Sensor filamento | Pin presente; irá un sensor **fotosensible** (lógica `HIGH`) |
| Volumen | 100 × 100 × 100 mm |
| Steps/mm | X606 Y606 Z600 E1040 |

> **Corriente**: ~450–500 mA bastan para mover los ejes a baja velocidad/aceleración;
> 800–1000 mA recomendado para imprimir. Para homing se prueba desde 350 mA.

## Estrategia de homing (sin endstops)

La K9 hace **crash homing** (homing a tope físico, sin sensor): G28 asume que está en
el máximo, mueve los ejes en negativo la distancia completa, y al llegar a 0 los ejes
chocan contra el marco; los motores patinan pasos y queda garantizado el cero.

Mecanismo en Marlin: con **`VALIDATE_HOMING_ENDSTOPS` desactivado**, un G28 que nunca
dispara un endstop **no mata** la impresora (ver `endstops.cpp::validate_homing_move`):
completa el movimiento y asume home. Con los pines `open` (sin switch) eso da el crash homing.

- **X / Y** → crash homing a tope. Para que el choque no sea destructivo se **baja la
  corriente** durante el homing (ver wrapper abajo).
- **Z** → homing con la **sonda** (`USE_PROBE_FOR_Z_HOMING` + `Z_SAFE_HOMING`), preciso y
  habilita la nivelación automática (UBL ya activo).

⚠️ **`HOMING_FEEDRATE_MM_M`** debe ser BAJO (30/30/6 mm/s) — velocidad alta = choque destructivo.

### Wrapper de corriente para homing (start-gcode del slicer)

No hay reducción de corriente automática por G28 para drivers PWM (eso es solo para TMC),
así que se hace por G-code:

```gcode
M907 X350      ; XY a 350 mA para choque suave (XY comparten canal; subir a 450 si no mueven)
G28            ; X/Y chocan a tope, Z baja con la sonda
M907 X1000     ; restaura corriente para imprimir
G29            ; (opcional) sondea malla UBL
```

## Flujo de configuración: `Marlin/config.ini`

El firmware se configura por **overrides en `Marlin/config.ini`** (ya enganchado en
`platformio.ini` → `extra_configs`). El build aplica estos deltas ENCIMA de
`config/EasyThreeD/ET4000PLUS/Configuration.h`. Así se cambia lo mínimo y casi todo lo
demás se ajusta **en caliente** (M-codes + `M500`) sin reflashear.

> `ini_use_config = base` → aplica SOLO la sección `[config:base]`.
> **NO usar `all`**: el fichero traía plantillas de ejemplo (RAMPS, cama 200×200…) que
> machacarían la config del K9.

Deltas actuales (ver `Marlin/config.ini`):

| Opción | Valor | Motivo |
|---|---|---|
| `use_probe_for_z_homing` | on | Z homing con sonda → habilita UBL |
| `z_safe_homing` | on | Sondea en el centro tras posicionar XY |
| `z_min_probe_endstop_inverting` | true | Polaridad de la sonda (verificado en vivo) |
| `nozzle_to_probe_offset` | `{ -10, 25, -2 }` | **ESTIMADO**, calibrar Z real |
| `validate_homing_endstops` | off | Permite el crash homing de X/Y |
| `homing_feedrate_mm_m` | `{30*60,30*60,6*60}` | Seguridad: choque suave |
| `filament_runout_sensor` | on | Futuro sensor fotosensible |
| `fil_runout_state` | HIGH | Lógica invertida |
| `marlin_dev_mode` | off | Firmware de producción |

## Compilar y flashear (minimizando flasheos)

1. Commit + push → **GitHub Actions** compila el `.bin` (`autoBuildAndUpload.yaml`).
   Si el `config.ini` tiene un error, el build falla ahí, **antes de flashear**.
2. Descargar `config/EasyThreeD/ET4000PLUS/mksLite.bin` y flashear.
3. Ajustar todo lo posible **en caliente** (M-codes + `M500`) sin volver a flashear.
   Solo requieren reflasheo los cambios de compilación (features, pines, homing feedrate).

## Calibración de la sonda (post-flasheo)

La sonda es de **deploy manual** (se baja/sube el brazo a mano) y **altura de disparo
indeterminada** (brazo basculante), así que el offset Z se calibra empíricamente:

1. `M119` → la sonda debe dar `open` en reposo (tras el fix de polaridad).
2. Desplegar el brazo a mano.
3. Verificar con G38 (seguro con la polaridad correcta): subir Z y
   `G38.2 Z0 F60` → debe bajar y parar al tocar la cama.
4. Offset Z: `G28` → bajar Z con `G1` hasta que una hoja de papel roce → leer Z →
   `M851 Z-<valor>` → `M500`.
5. Afinar X/Y del offset si la malla queda descentrada (`M851 X.. Y..` + `M500`).
6. Malla: `G29` → `M500`. Recuerda recoger el brazo antes de imprimir.

> Estimación inicial del offset: **X−10 mm, Y+25 mm, Z−2 mm**.
> Opcional: `PAUSE_BEFORE_DEPLOY_STOW` hace que Marlin pause y pida desplegar/recoger
> la sonda manualmente (ideal para deploy manual).

## ⚠️ Reporte en tiempo real — `FULL_REPORT_TO_HOST_FEATURE` rompe los hosts estándar

`Configuration_adv.h` activa, bajo `REALTIME_REPORTING_COMMANDS`, el feature
**`FULL_REPORT_TO_HOST_FEATURE`**: Marlin auto-reporta su estado en formato Grbl/CNC
(`<Idle|MPos:0.000,0.000,0.000|...>`) de forma **continua y no solicitada**.

Los hosts de impresión 3D estándar hablan el **protocolo de líneas de Marlin** (esperan un
`ok` por comando y parsean `X:.. Y:.. Z:.. Count` / `T:.. B:..`). **No entienden** el flujo
Grbl `<...>` y el flood **ROMPE la conexión**:

- **OctoPrint**: la impresora puede no conectar, colgarse, dar timeout o quedar sin
  responder **justo tras flashear** (observado en esta placa).
- **Pronterface / Repetier / Cura / la mayoría**: mismo problema.

**Mantenerlo activo SOLO si el host consume y suprime los `<...>`.** En OctoPrint eso es un
plugin que engancha `octoprint.comm.protocol.gcode.received`, parsea cada `<...>` y devuelve
`""` para que OctoPrint no lo procese (lo hace el plugin **OctoPrint Boost Kit / k9suite**, que
además usa la posición en vivo para animar el homing/probing en su visor 3D).

**Si tu host falla / no conecta tras flashear:**

1. Comenta el feature: `//#define FULL_REPORT_TO_HOST_FEATURE`, recompila y reflashea.
2. No pierdes posición en vivo: `REALTIME_REPORTING_COMMANDS` (`S000`/`P000`/`R000`) y
   **`AUTO_REPORT_POSITION`** (`M154 S<seg>`, formato Marlin, **seguro para el host**) siguen
   disponibles y bastan para sincronizar la posición **sin** el flood Grbl.
3. Recuperación: hay un `mksLite.bin` conocido-bueno SIN este feature en el historial git
   (commits `revert: ... FULL_REPORT_TO_HOST_FEATURE`).

---

## Nivelación 7×7 + 4 medidas + slots UBL (Boost Kit, 2026-06)

- **Grid UBL 5×5 → 7×7**: `config/EasyThreeD/ET4000PLUS/Configuration.h`
  (`GRID_MAX_POINTS_X 7`, bloque `AUTO_BED_LEVELING_UBL`).
- **4 medidas por punto**: `Marlin/config.ini` → `multiple_probing=3`, `extra_probing=1`
  (TOTAL_PROBING=4: 4 lecturas, descarta el outlier, promedia 3).
  La vía **plugin** (Boost Kit) sondea con G30 ×4 y calcula la **moda**; la vía
  **G29 nativa** (sin plugin) usa el promedio/mediana de Marlin.
- **Slots UBL**: `EEPROM_SETTINGS` y `UBL_SAVE_ACTIVE_ON_M500` ya estaban activos.
  El plugin guarda con `G29 S<slot>` + `M500` y activa con `G29 L<slot>` + `M420 S1`.
  El plugin gestiona el **nombre→índice** y guarda copia de los Z para visualizar.
- **VERIFICAR tras flashear**: `M503` debe reportar el grid 7×7 y UBL debe listar
  **≥4 slots** (`G29 L`). Con 7×7 (49 pts × 4 B ≈ 196 B/malla) deben caber 4 en la
  EEPROM emulada del STM32F103; si no, reducir slots usados en el plugin.
- Área imprimible se mantiene en **100×100** (ejes); la cama real 120×120 y el margen
  de 10mm son solo representacionales en el plugin (visor de gcode + visor topográfico).
