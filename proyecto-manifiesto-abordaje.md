# Manifiesto de Abordaje — Escáner de Pases (Satena)

Web ligera de una sola página (HTML+CSS+JS, sin frameworks ni instalación) para escanear con la cámara del celular los códigos de barras PDF417 de los pases de abordar, extraer automáticamente los datos del pasajero y armar el listado de abordaje evitando duplicados.

**Archivo:** `manifiesto-abordaje.html` (autocontenido, ~34 KB de código propio + 1 librería externa cargada por CDN).

**Última actualización:** 18 de septiembre de 2026 — corrección del escáner de cámara (ver sección 3.1 y 6).

---

## 1. Objetivo

- Correr en cualquier navegador Android (Chrome) sin instalar nada.
- Usar la cámara trasera para leer el código PDF417 del pase.
- Extraer: nombre del pasajero, código de reserva (PNR), ruta, aerolínea + vuelo, asiento y número de tiquete.
- Agregar cada pasajero a una lista **sin repetir** al mismo pasajero dos veces.
- Conservar la lista aunque se cierre o recargue el navegador.
- Exportar el listado final en CSV.

---

## 2. Formato de datos: estándar IATA BCBP

Los pases de abordar codifican la información en texto plano de ancho fijo, según el estándar **IATA Bar Coded Boarding Pass (BCBP)**. Ejemplo real usado para construir y probar este proyecto:

```
M1ZAPATA CAICEDO/CLARAELHCSOC EOHLPZ9R 8839 259Y003A0003 148>3180WW6258B9R              2A019243621098600
```

### Campos obligatorios (posición fija — 100% confiables)

| Posición (0-indexada) | Longitud | Campo | Valor en el ejemplo |
|---|---|---|---|
| 0 | 1 | Código de formato | `M` |
| 1 | 1 | Número de tramos (legs) | `1` |
| 2–22 | 20 | Nombre del pasajero (`APELLIDOS/NOMBRES`) | `ZAPATA CAICEDO/CLARA` |
| 22 | 1 | Indicador de tiquete electrónico | `E` |
| 23–30 | 7 | PNR / localizador de reserva | `LHCSOC` |
| 30–33 | 3 | Aeropuerto de origen | `EOH` |
| 33–36 | 3 | Aeropuerto de destino | `LPZ` |
| 36–39 | 3 | Aerolínea operadora | `9R` |
| 39–44 | 5 | Número de vuelo | `8839` |
| 44–47 | 3 | Fecha de vuelo (día juliano) | `259` |
| 47 | 1 | Clase / compartimento | `Y` |
| 48–52 | 4 | Asiento (fila + letra) | `003A` → `3A` |
| 52–57 | 5 | Secuencia de check-in | `0003` |
| 57 | 1 | Estado del pasajero | `1` |

Estos campos siguen el estándar al pie de la letra y se parsean cortando el texto por posición fija — no dependen de heurísticas.

### Número de tiquete (⚠️ heurística, no 100% garantizada)

Después de la posición 58 viene una sección de **longitud variable** (versión, aerolínea emisora, número de tiquete, código de selección, etc.) cuya estructura completa depende de cómo cada aerolínea la haya generado. El estándar IATA no fija una posición universal para el número de tiquete dentro de esa sección.

**Lo que hace esta app:** busca la última racha de 10 o más dígitos seguidos en esa sección y toma los primeros 13 (código numérico de aerolínea + número de serie). En el ejemplo real de Satena esto extrae correctamente `0192436210986`.

**Recomendación:** antes de confiar en este dato para trámites oficiales, prueba con 2–3 pases reales más de Satena para confirmar que el patrón se mantiene. La app marca internamente el campo `ticketConfidence` (`heuristica` / `parcial` / `none`) por si se necesita depurar.

---

## 3. Arquitectura técnica

- **Un solo archivo HTML** con CSS y JavaScript embebidos — nada que compilar, nada que instalar, carga rápido (cumple el requisito de "web ligera").
- **Cámara:** la app abre el stream ella misma con `navigator.mediaDevices.getUserMedia`, pidiendo cámara trasera (`facingMode: 'environment'`) **y alta resolución** (ver 3.1).
- **Parseo:** función `parseBCBP()` en JavaScript puro, con los offsets de la tabla anterior.
- **Deduplicación:** cada pasajero se guarda con una llave única = número de tiquete (si se pudo extraer) o, si no, `PNR + aerolínea + vuelo + nombre`. Antes de agregar un pasajero se revisa si esa llave ya existe en la lista.
- **Persistencia:** `localStorage` del navegador — el listado sobrevive a cerrar la pestaña o apagar la pantalla del teléfono.
- **Exportar CSV:** descarga directa cuando el entorno lo permite; si no, cae automáticamente a copiar el CSV al portapapeles (o a un cuadro de texto para copiar manualmente) como respaldo.

### 3.1 Decodificación del código de barras (corregido)

Esta es la parte que fallaba en la primera versión y que se rehízo.

**Dos motores, en este orden:**

1. **`BarcodeDetector` nativo del sistema** — API del navegador que delega en el decodificador del sistema operativo. Está disponible en Android (requiere Google Play Services), macOS y ChromeOS, e incluye `pdf417` entre sus formatos. Es notablemente más rápido y tolerante que un decodificador en JavaScript. La app detecta en tiempo de ejecución si existe y si soporta PDF417 (`BarcodeDetector.getSupportedFormats()`); solo entonces lo usa.
2. **ZXing** (`@zxing/library@0.21.3`, CDN `cdn.jsdelivr.net`) — respaldo automático si el navegador no tiene el motor nativo o no soporta PDF417.

**Resolución de video — la causa raíz del fallo original.** La versión anterior llamaba a `decodeFromVideoDevice()` de ZXing, que internamente pide solo `{ video: { facingMode: 'environment' } }`, sin ninguna restricción de tamaño. El navegador entonces entrega la resolución por defecto, que en teléfonos Android suele ser **640×480**. A esa resolución las barras de un PDF417 de pase de abordar no se distinguen: la cámara se ve nítida en pantalla, pero el decodificador no encuentra nada nunca. Ese era exactamente el síntoma de "no ocurre nada al apuntar al código".

Ahora la app pide resolución explícitamente, en cascada de mayor a menor, quedándose con la primera que el dispositivo acepte:

| Intento | Restricciones |
|---|---|
| 1 | `facingMode: {exact:'environment'}`, 2560×1440 ideal |
| 2 | `facingMode: {ideal:'environment'}`, 1920×1080 ideal |
| 3 | `facingMode: 'environment'` (sin tamaño) |
| 4 | `video: true` (cualquier cámara) |

Si el usuario niega el permiso, la cascada se detiene de inmediato y se muestra el mensaje correspondiente (no se reintenta en bucle).

**Ayudas de enfoque y encuadre añadidas:**

- `focusMode: 'continuous'` si el dispositivo lo reporta como capacidad.
- Control deslizante de **zoom** (solo si la cámara lo soporta) — acercarse ópticamente mejora mucho la lectura de códigos densos.
- Botón **📷**: captura una foto a resolución de sensor (`ImageCapture.takePhoto()`, con `grabFrame()` y luego un cuadro del video como respaldos) y la analiza. Es el plan B cuando el video en vivo no alcanza.
- **Línea de diagnóstico** permanente en la pantalla del escáner: motor activo, resolución real del video y número de análisis realizados. Sirve para saber al instante si el problema es el permiso, la resolución o el decodificador.

**Contexto seguro (requisito de navegador).** `getUserMedia` solo existe en páginas servidas por `https://` o `localhost`. Si el archivo se abre directamente desde el almacenamiento del teléfono (`file:///storage/...`), Chrome no expone la cámara y la app no puede hacer nada al respecto. La versión corregida detecta ese caso y lo dice explícitamente en pantalla, en vez de quedarse callada.

### 3.2 Flujo de uso

1. El agente toca **"Escanear pase de abordar"** → se pide permiso de cámara.
2. Se abre la cámara trasera en la resolución más alta disponible y el motor elegido analiza los cuadros buscando un PDF417.
3. Al detectar un código, se parsea y se calcula la llave de duplicado.
   - Si es nuevo → se agrega a la lista, vibra el teléfono y muestra confirmación verde.
   - Si ya estaba → muestra aviso naranja "Ya estaba en la lista", no lo duplica.
   - Si el texto no tiene forma de BCBP válido → aviso indicando que el código sí se leyó pero su contenido no es un pase.
4. Tras cada lectura hay una pausa de ~1.7 s antes de volver a escanear, para evitar lecturas repetidas del mismo pase.
5. La lista, contador, buscador y exportar CSV están siempre visibles en la pantalla principal.

---

## 4. Funciones incluidas

- Escaneo continuo con cámara trasera en alta resolución.
- Selección automática del mejor motor de decodificación disponible.
- Linterna (torch) on/off, si el dispositivo la soporta.
- Zoom de cámara, si el dispositivo lo soporta.
- Captura de foto a máxima resolución para forzar una lectura difícil.
- Línea de diagnóstico en pantalla (motor, resolución, análisis).
- Ingreso manual de respaldo (pegar el texto crudo del código si el pase está dañado o no escanea).
- Buscador de pasajeros ya agregados.
- Eliminar un pasajero individual (por si se escaneó por error).
- Vaciar toda la lista (con confirmación) para empezar un nuevo vuelo/turno.
- Campo editable de "Vuelo" (para identificar la sesión y nombrar el CSV exportado).
- Exportar CSV con: nombre completo, apellidos, nombres, PNR, origen, destino, aerolínea, vuelo, clase, asiento, número de tiquete y hora de escaneo.
- Diseño adaptado a móvil, con modo claro/oscuro automático según el sistema.

---

## 5. Personalización

### Diccionario de aeropuertos

Satena cubre aeródromos regionales colombianos que muchas veces no están en bases de datos IATA genéricas. Dentro del HTML, al inicio del `<script>`, está este diccionario editable:

```js
var AIRPORTS = {
  EOH: "Olaya Herrera (Medellín)",
  LPZ: "Los Pozos (San Gil)"
  // Agrega aquí más rutas propias, ej: "ADZ": "Providencia"
};
```

Se puede seguir agregando cualquier código de 3 letras que use Satena internamente.

### Clases / compartimentos

```js
var CLASSES = { Y:"Económica", C:"Ejecutiva", F:"Primera" };
```

---

## 6. Validación realizada

### 6.1 Pruebas de la primera versión

- Se generó un PDF417 limpio con el texto exacto del ejemplo y se decodificó con la misma versión de ZXing usada en la web → coincidió carácter por carácter con el original.
- Se probó el parser contra el ejemplo real: nombre, PNR, ruta, vuelo, asiento y ticket salieron correctos.
- Se probó la interfaz completa en un navegador real (Playwright): agregar pasajero, detectar duplicado y rechazar un código inválido funcionaron correctamente.
- Se encontró y corrigió un bug real de CSS que impedía ocultar correctamente el modal y el visor de cámara.

### 6.2 Diagnóstico y prueba de la corrección del escáner

Se montó una prueba de extremo a extremo: se generó el PDF417 del pase de ejemplo, se convirtió en un video y se le entregó a un Chrome real como **cámara falsa** (`--use-file-for-fake-video-capture`), sirviendo la página por HTTP local. Resultado con el **código original**, variando únicamente la resolución del stream:

| Resolución de la cámara | Resultado |
|---|---|
| 1920×1080 | decodificó de inmediato (primer cuadro analizado) |
| 640×480 | 15 segundos apuntando al código, **cero lecturas** |

Esto confirma el diagnóstico: el decodificador y el parser siempre estuvieron bien; lo que faltaba era resolución. Con la versión corregida, la misma prueba decodifica en el primer análisis y muestra `Clara Zapata Caicedo · 9R 8839 · EOH→LPZ · Asiento 3A · PNR LHCSOC · Tkt 0192436210986`.

También se reverificó, sobre la versión corregida y sin errores de JavaScript en consola: alta por ingreso manual, detección de duplicado, rechazo de texto que no es BCBP, persistencia tras recargar la página, botón de foto y cierre limpio del escáner (se apaga la cámara).

### 6.3 Lo que sigue sin poder probarse aquí

- **Escaneo con cámara física real sobre un pase impreso.** El entorno de desarrollo no tiene cámara; la prueba usa un video sintético perfecto (barras nítidas, sin reflejos, sin desenfoque, sin curvatura del papel). El comportamiento con papel térmico real, luz de mostrador y enfoque de un celular concreto solo se confirma probándolo.
- **El motor nativo `BarcodeDetector`**, que es el camino principal en Android, no existe en el Chrome de Linux usado para las pruebas. Toda la validación anterior se hizo por la ruta de respaldo (ZXing). La ruta nativa usa la API pública documentada y hace detección de capacidades en tiempo de ejecución, pero su rendimiento real solo se verá en el teléfono.

---

## 7. Limitaciones conocidas

- La cámara **exige `https://` o `localhost`**. Abrir el `.html` descargado con `file://` nunca dará cámara, por diseño del navegador.
- El motor nativo en Android depende de Google Play Services; si falta, se usa ZXing, que es más lento y más exigente con el enfoque.
- Pensada para pases de **un solo tramo** (`M1`, un vuelo). Pases con conexiones (`M2`, `M3`…) no se han probado.
- El número de tiquete se extrae por heurística (ver sección 2), no por un offset garantizado por el estándar.
- Los nombres de aeropuertos deben completarse manualmente en el diccionario; solo trae precargados `EOH` y `LPZ`.
- No sincroniza entre varios celulares/agentes en tiempo real — cada dispositivo tiene su propia lista local.

---

## 8. Solución de problemas

La línea de diagnóstico del escáner (debajo del mensaje "Apunta al código…") indica dónde está el problema:

| Lo que ves | Qué significa | Qué hacer |
|---|---|---|
| Mensaje sobre `file://` | La página no se está sirviendo por HTTPS | Publicarla en una URL `https://` o servirla con un servidor local |
| "Permiso de cámara denegado" | El sitio tiene el permiso bloqueado | Ajustes del navegador → Permisos del sitio → Cámara |
| Video: `640x480` o similar | El teléfono no concedió alta resolución | Cerrar otras apps que usen la cámara; probar el botón 📷 |
| Motor: `ZXing (respaldo)` en Android | No hay `BarcodeDetector` nativo | Verificar/actualizar Google Play Services y Chrome |
| Análisis sube pero no lee | Llega imagen y se analiza; el código no se resuelve | Acercarse, usar zoom, encender la linterna, aplanar el papel, evitar reflejos; si no, botón 📷 |
| Análisis no sube de 0 o 1 | El bucle de análisis no avanza | Cerrar y volver a abrir el escáner |
| "Código leído, pero no es un pase" | Se decodificó algo que no empieza por `M1` | Es otro código de barras del pase; apuntar al PDF417 grande |

Como último recurso siempre está el botón **Manual**, que acepta el texto crudo del BCBP pegado a mano.

---

## 9. Próximos pasos posibles (si se necesitan)

- Prueba en teléfono real con pases impresos de Satena y ajuste fino de zoom/resolución según lo que reporte la línea de diagnóstico.
- Sincronización en tiempo real entre varios dispositivos del mismo mostrador.
- Soporte para pases con más de un tramo.
- Manifiesto de aeropuertos ampliado y compartido entre varios usuarios.
- Registro con foto/firma de respaldo para embarque manual.
