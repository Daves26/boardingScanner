# Manifiesto de Abordaje — Escáner de Pases (Satena)

Web ligera de una sola página (HTML+CSS+JS, sin frameworks ni instalación) para escanear con la cámara del celular los códigos de barras PDF417 de los pases de abordar, extraer automáticamente los datos del pasajero y armar el listado de abordaje evitando duplicados.

**Archivo:** `manifiesto-abordaje.html` (autocontenido, ~25 KB de código propio + 1 librería externa cargada por CDN).

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
- **Decodificación de código de barras:** librería [ZXing](https://github.com/zxing-js/library) (`@zxing/library@0.21.3`), cargada desde CDN público (`cdn.jsdelivr.net`). Es la misma librería estándar de código abierto usada en la mayoría de escáneres web de códigos de barras; soporta PDF417 de forma nativa.
- **Cámara:** `ZXing.BrowserPDF417Reader` + `navigator.mediaDevices.getUserMedia`, pidiendo específicamente la cámara trasera (`facingMode: 'environment'`).
- **Parseo:** función `parseBCBP()` en JavaScript puro, con los offsets de la tabla anterior.
- **Deduplicación:** cada pasajero se guarda con una llave única = número de tiquete (si se pudo extraer) o, si no, `PNR + aerolínea + vuelo + nombre`. Antes de agregar un pasajero se revisa si esa llave ya existe en la lista.
- **Persistencia:** `localStorage` del navegador — el listado sobrevive a cerrar la pestaña o apagar la pantalla del teléfono.
- **Exportar CSV:** usa la capacidad de descarga del entorno de artefactos de Claude; si no está disponible, cae automáticamente a copiar el CSV al portapapeles (o a un cuadro de texto para copiar manualmente) como respaldo.

### Flujo de uso

1. El agente toca **"Escanear pase de abordar"** → se pide permiso de cámara.
2. La cámara trasera se activa y ZXing analiza los cuadros de video buscando un PDF417.
3. Al detectar un código, se parsea y se calcula la llave de duplicado.
   - Si es nuevo → se agrega a la lista, vibra el teléfono y muestra confirmación verde.
   - Si ya estaba → muestra aviso naranja "Ya estaba en la lista", no lo duplica.
   - Si el texto no tiene forma de BCBP válido → aviso de "código no reconocido".
4. Tras cada lectura hay una pausa de ~1.7 s antes de volver a escanear, para evitar lecturas repetidas del mismo pase.
5. La lista, contador, buscador y exportar CSV están siempre visibles en la pantalla principal.

---

## 4. Funciones incluidas

- Escaneo continuo con cámara trasera.
- Linterna (torch) on/off, si el dispositivo la soporta.
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

Antes de entregar la app se hicieron pruebas reales (no solo revisión de código):

- Se generó un PDF417 limpio con el texto exacto del ejemplo y se decodificó con la misma versión de ZXing usada en la web → coincidió carácter por carácter con el original.
- Se probó el parser contra el ejemplo real: nombre, PNR, ruta, vuelo, asiento y ticket salieron correctos.
- Se probó la interfaz completa en un navegador real (Playwright): agregar pasajero, detectar duplicado y rechazar un código inválido funcionaron correctamente.
- Se encontró y corrigió un bug real de CSS que impedía ocultar correctamente el modal y el visor de cámara.

**No se pudo probar en este entorno:** el escaneo con cámara en vivo sobre un dispositivo Android real, ya que el entorno de desarrollo no tiene cámara física. El código de cámara usa la API pública y documentada de ZXing, pero esa parte específica solo quedará 100% confirmada al probarla en un teléfono real.

---

## 7. Limitaciones conocidas

- Pensada para pases de **un solo tramo** (`M1`, un vuelo). Pases con conexiones (`M2`, `M3`…) no se han probado.
- El número de tiquete se extrae por heurística (ver sección 2), no por un offset garantizado por el estándar.
- Los nombres de aeropuertos deben completarse manualmente en el diccionario; solo trae precargados `EOH` y `LPZ`.
- No sincroniza entre varios celulares/agentes en tiempo real — cada dispositivo tiene su propia lista local.

---

## 8. Próximos pasos posibles (si se necesitan)

- Sincronización en tiempo real entre varios dispositivos del mismo mostrador.
- Soporte para pases con más de un tramo.
- Manifiesto de aeropuertos ampliado y compartido entre varios usuarios.
- Registro con foto/firma de respaldo para embarque manual.
