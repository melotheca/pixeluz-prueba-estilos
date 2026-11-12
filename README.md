# Prueba de estilos Johnson (Pixeluz) — página web

Página mínima para probar los modelos Johnson (.tflite) subiendo una foto,
sin usar la app todavía. Todo corre dentro del navegador con TensorFlow.js;
la foto nunca sale del dispositivo.

## Archivos

- `index.html` — la página (input de foto, 2 botones de modelo, resultado, descarga).
- `models/johnson_vangogh_fp32.tflite` — Van Gogh, 6,761,220 bytes.
- `models/johnson_ukiyoe_fp32.tflite` — Ukiyo-e, 6,760,860 bytes.
- `.nojekyll` — para que GitHub Pages sirva los archivos tal cual.

## Cómo funciona

- Los modelos solo aceptan 256x256. La página parte la foto en tiles de 256
  con solape de 64 y mezcla lineal, con la misma matemática que la
  verificación local (`verify_ukiyoe_tflite.py`): posiciones de tile,
  rampa `linspace(0,1,64)` en los bordes, reflect-pad en los bordes de la
  foto. Verificado: el código JS de la página da **exactamente** el mismo
  resultado que la referencia en Python (con modelo identidad) en 6 tamaños,
  incluidos 1404x1120 y 1900x1060 (diff máxima 0.0).
- Carga los modelos con `@tensorflow/tfjs-tflite@0.0.1-alpha.10` y
  `@tensorflow/tfjs@4.22.0` desde jsDelivr. Importante: la página fija el
  WASM con `setWasmPath(.../wasm/)` porque los .wasm **no** están en `dist/`
  (ahí dan 404); sin eso el modelo no carga.
- Las fotos se reducen a 2048 px de lado mayor como máximo para que el
  navegador no se cuelgue (una foto de 1404x1120 son 42 tiles, unos segundos).

## Despliegue en GitHub Pages

1. Crear la cuenta en https://github.com/signup con el correo
   `melotheca@gmail.com` (nombre de usuario sugerido: `melotheca`).
   GitHub pide CAPTCHA, código de verificación por correo y contraseña:
   eso solo lo puede hacer el usuario en persona.
2. Crear un repositorio público nuevo, por ejemplo `pixeluz-prueba-estilos`.
3. Subir todo el contenido de esta carpeta (o `git push` con `gh`).
4. Settings → Pages → Build and deployment → Deploy from a branch →
   rama `main`, carpeta `/ (root)`. Guardar.
5. La página queda en `https://<usuario>.github.io/pixeluz-prueba-estilos/`
   (por ejemplo, con usuario `melotheca`:
   `https://melotheca.github.io/pixeluz-prueba-estilos/`).

## Estado de las pruebas (honesto)

- La lógica de tiling de la página está verificada contra Python: idéntica.
- El HTML y los scripts CDN responden 200; los modelos se sirven bien.
- **El modelo NO se ha ejecutado todavía en un runtime JS en este entorno.**
  `tfjs-tflite` es una librería solo de navegador: en Node falla porque su
  loader necesita `self`/`document`/`location` y un Web Worker con
  `importScripts` (el .wasm sí se descarga, pero el módulo nunca inicializa).
  En este entorno no hay Chromium headless. La primera prueba real del
  modelo en el navegador debe hacerse tras el despliegue, en Chrome o
  Firefox: si algo falla, la página muestra el error en pantalla.
- Operaciones de los modelos (ambos iguales, 172 ops): ADD, CONV_2D, MUL,
  SUB, TANH, MEAN, RSQRT, RESIZE_NEAREST_NEIGHBOR, SQUARED_DIFFERENCE,
  MIRROR_PAD. Sin ops personalizadas, sin TRANSPOSE, sin FP16.
  `tfjs-tflite` ejecuta el intérprete TFLite real en WASM, así que estas
  operaciones estándar corren (las que XNNPACK no acelera caen al kernel
  de referencia). El riesgo es bajo, pero no está probado hasta correrlo
  en un navegador real. MIRROR_PAD (el ReflectionPad) y la descomposición
  de InstanceNorm (SUB + SQUARED_DIFFERENCE + MEAN + RSQRT + MUL) son los
  puntos a vigilar en esa primera prueba.
- GitHub Pages no envía cabeceras COOP/COEP, así que el WASM no usa
  multi-hilo (usa la variante SIMD). Funciona, solo más lento.
- Detalle conocido, igual que en la verificación en Python: la primera
  fila y la primera columna del resultado salen en negro (la rampa de
  mezcla vale 0 en el borde). Es 1 px, invisible a simple vista.
