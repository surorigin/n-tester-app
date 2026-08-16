# N-Tester Lectura Automática

PWA (app web instalable) que se conecta por Bluetooth al medidor **Yara N-Tester BT**, calcula un valor estimado de nitrógeno foliar en tiempo real, y arma mapas de zonas por lote.

**App en vivo:** `https://TU-USUARIO.github.io/n-tester-app/`

---

## Qué hace

- Se conecta al N-Tester por Web Bluetooth (Chrome/Android) y lee la señal óptica cruda del sensor
- Detecta solo cuando bajás la pinza y se toma una lectura nueva (sondeo automático, sin botones)
- Calcula un **N estimado** con una fórmula propia, calibrada contra la app oficial YaraPlus
- Organiza las lecturas por **sesión** (Campo + Lote), sin mezclar relevamientos distintos
- Georeferencia cada lectura con el GPS del teléfono
- Permite cargar el **límite del lote** (importando un KML o dibujándolo en el mapa)
- Genera un **mapa de zonas interpolado** (rojo → amarillo → verde) dentro del límite
- Soporta una **parcela de referencia / franja de suficiencia de N**: tomás varias lecturas en una zona bien fertilizada, la app promedia esa referencia, y clasifica el resto del lote como Óptimo / Marginal / Déficit según el % respecto a esa referencia
- Guarda todo localmente en el teléfono (localStorage) — persiste aunque cierres la app
- Exporta cada sesión como CSV o KML (puntos + límite)
- Suena un beep al capturar cada lectura, y mantiene la pantalla encendida durante la sesión (Wake Lock)

## Cómo se llegó a esto (por si hace falta retocar)

### El problema de partida
El N-Tester de fábrica se conecta a la app YaraPlus, que muestra el resultado de nitrógeno pero no expone ese dato crudo a otras apps.

### Cómo se encontró el protocolo BLE
Se usó **nRF Connect** (app de diagnóstico Bluetooth genérica) para listar todos los servicios GATT del equipo. Se identificaron 4 servicios propietarios de Yara (no estándar):

```
00006e10-0000-1000-8000-00805f9b34fb
00006e20-0000-1000-8000-00805f9b34fb
00006e30-0000-1000-8000-00805f9b34fb
00006e40-0000-1000-8000-00805f9b34fb
```

Dentro de esos servicios, dos características (`float32`, little-endian) resultaron ser las señales ópticas activas del sensor — cambian con cada medición real:

```
Característica 6e11: 00006e11-0000-1000-8000-00805f9b34fb
Característica 6e12: 00006e12-0000-1000-8000-00805f9b34fb
```

Las demás características cercanas (`6e13`, `6e14`, `6e15`) resultaron ser constantes de calibración de fábrica, casi fijas entre mediciones — no se usan en el cálculo.

### La fórmula de estimación
No existe documentación pública de la fórmula exacta que usa Yara. Se dedujo por **regresión**, comparando la salida cruda del sensor contra el valor real que mostraba YaraPlus para las mismas hojas:

```
N_estimado ≈ 478 × ln(6e11 / 6e12) − 449
```

Calibrada con 9 pares de mediciones reales (rango ~480-1200). Error típico observado: 45-80 unidades. **No es el valor oficial de Yara** — es una aproximación por ingeniería inversa. Cuantos más pares de comparación (crudo vs. YaraPlus) se sumen, más se puede afinar reajustando esta fórmula.

### Detección automática de lectura
La app sondea (`readValue()`) las características `6e11`/`6e12` cada ~400ms. Cuando dos sondeos seguidos dan un valor casi idéntico (lectura "asentada") y ese valor es distinto a la última lectura ya mostrada, se interpreta como una medición nueva y se captura sola, sin que el usuario toque nada.

### Mapa de zonas
Interpolación por **IDW** (ponderación por distancia inversa, potencia 2) sobre una grilla dentro del límite del lote. Sin parcela de referencia, los 5 colores se reparten por rango relativo (mín-máx de esa sesión). Con parcela de referencia, se usan umbrales agronómicos fijos sobre el índice de suficiencia (SI = lectura/referencia × 100): <80% / 80-90% / 90-95% / 95-100% / ≥100%.

---

## Archivos del proyecto

| Archivo | Qué es |
|---|---|
| `index.html` | Toda la app (HTML + CSS + JS en un solo archivo) |
| `manifest.json` | Metadata para que sea instalable como PWA |
| `sw.js` | Service worker, cachea la app para que funcione offline |
| `icon-192.png`, `icon-512.png` | Íconos de la app instalada |

## Cómo actualizar la app

1. Editar `index.html` (o pedirle a Claude que lo edite)
2. Subir el archivo al repo de GitHub, reemplazando el anterior
3. En el celular: cerrar la app instalada del todo y volver a abrirla (el service worker cachea, así que a veces tarda un momento en tomar la versión nueva)

## Ideas pendientes / próximos pasos

- **App nativa Android (APK):** Web Bluetooth no existe en un WebView nativo (Capacitor/Cordova), así que no alcanza con "empaquetar" esta PWA — hay que reescribir la conexión BLE con un plugin nativo. Viable con Claude Code corriendo en una compu con Android Studio/SDK instalado (no se puede hacer desde este chat).
- **Refinar la fórmula de N** a medida que se sumen más pares de comparación contra YaraPlus, sobre todo en los extremos del rango (muy bajo / muy alto).
- **Elegir qué polígono importar** si un KML trae varios lotes — hoy la app toma siempre el primero que encuentra.
- Posible: exportar un reporte comparando distintas sesiones del mismo lote a lo largo del tiempo (evolución).

## Aviso importante

El valor de N que muestra esta app es una **estimación por ingeniería inversa**, no el valor certificado de Yara. Sirve como referencia de campo, pero no reemplaza mediciones oficiales para decisiones de fertilización de alto impacto económico sin contrastar periódicamente contra la app oficial.
