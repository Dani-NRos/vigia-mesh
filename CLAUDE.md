# Vigía servidor (camino de la 2.0)

> **Esta rama (`mapa`) ya no se usa**: el desarrollo sigue en el repositorio `Dani-NRos/talaia-hub` (Talaia Hub), con
> su historia y su propio CLAUDE.md. Lo de abajo es como estaba al pasar allí.

**Vigía servidor**: vigilancia 24/7 en Linux (BMAX con Ubuntu, Raspberry Pi 4/5, Pi Zero 2 W) de nodos Meshtastic y
dispositivos MQTT, con histórico en SQLite, Home Assistant por MQTT y acceso desde fuera con Tailscale. Esta rama
(`mapa`) es la de la **versión 2.0** (plan abajo): tendrá nombre nuevo y repositorio propio.

Ramas del repositorio `vigia-mesh`:
- `main`: la **app ligera** (PWA de Bluetooth y cable) **congelada en la 1.20**, publicada en GitHub Pages
  (https://dani-nros.github.io/vigia-mesh/). Solo arreglos graves. No se renombra el repositorio: las URL de GitHub
  Pages de un proyecto no se redirigen al renombrarlo y las apps instaladas dejarían de actualizarse. Por eso el
  servidor se irá a un repositorio nuevo cuando tenga nombre.
- `version-1.20`: el servidor y la app tal como estaban en la 1.20 (el proxy de esta sesión no deja subir etiquetas).
- `mapa`: desarrollo de la 2.0 (solo el servidor).

## Normas de trabajo (de Dani)

- **Nada por la malla sin preguntar**: ni mensajes, ni avisos de prueba, ni traceroutes, ni peticiones a nodos.
  Tampoco cambiar sus alarmas o sus nodos vigilados. Los selectores de Bluetooth y de puerto los maneja él.
- **Una cosa cada vez**: terminar, probar y comprobar cada fase antes de seguir.
- **Pruebas** automáticas con `node --test`; cuando haga falta, con su equipo real (lo prueba él).
- **Arreglos en la app ligera** (rama `main`): primero una copia de pruebas en `/beta/`, luego publicar.
- **Idioma**: textos de la app en español de España, claros y sin jerga (hasta que llegue el inglés con el rediseño).
  Desde la 2.0: código nuevo, comentarios nuevos y mensajes de commit en inglés.
- **Sin complacencia**: si algo no tiene sentido o hay una opción mejor, decirlo; preguntar al decidir algo.

## Estructura

```
codigo/                 raíz de npm workspaces: npm install, npm test, npm run build
  nucleo/               lógica sin pantalla, la usan la app y el servidor (@vigia/nucleo)
    src/motor.js        nodos, lecturas, alarmas, peticiones, traceroute, avisos por la malla, lo que llega del receptor
    src/acciones.js     lo que cambia el usuario (vigilar, límites, valla, ajustes…): igual en la app y el servidor
    src/receptor-base.js  ciclo de conexión, lectura inicial, reintentos y peticiones con id (app y servidor)
    src/textos.js       tiempos y textos comunes (dur, ago, stamp…)
    src/canales.js      qué canal es privado; recorte a 200 bytes
    src/tramas.js       tramas del puerto serie (0x94 0xC3), troceado propio, latido, despertar, fabricantes USB
    src/mqtt-datos.js   datos MQTT: tipos y unidades, qué parece cada tema, leer el valor de un JSON, comprobar la
                        configuración (también lo que puede salir a la malla); metrics.js registra sus métricas
    src/metrics.js trend.js delivery.js route.js geo.js pb.js
  app/                  la PWA (@vigia/app)
    src/app.js          pantalla; crea el motor con ganchos (redibujar, guardar en localStorage, sonar…) y radio.
                        Con window.__VIGIA_SERVIDOR (lo pone el servidor al servirla) arranca en modo «Servidor»:
                        estado del servidor por SSE y órdenes por POST (src/remoto.js)
    src/receptor.js     conexión al nodo desde el navegador (Bluetooth o Web Serial): de la app ligera
    src/ble.js serial.js store.js (IndexedDB) demo.js   también de la app ligera: se van con el rediseño
    src/map.js mapdata.js meshsearch.js remoto.js
    index.src.html build.mjs pwa/ (manifiesto, iconos y sw.js: la pantalla del servidor se instala como app)
  servidor/             Vigía servidor (@vigia/servidor), Node ≥ 22.13
    src/main.js         arranque (systemd o a mano); src/vigia.js: motor + receptor + SQLite + tareas periódicas
    src/receptor.js enlace.js puertos.js   nodo por USB (serialport, con binarios: no compila) o TCP 4403
    src/almacen.js      SQLite (node:sqlite): lecturas y el estado que la app guarda en localStorage
    src/web.js sincro.js  la app servida en la red de casa y su API (SSE + órdenes), sin contraseña (decisión de Dani)
    src/mqtt.js         MQTT (MQTT.js): varios brokers a la vez (una conexión cada uno), configurados en Ajustes y
                        guardados en SQLite; reconexión; testamento solo en el de Home Assistant; migra lo de la 1.16
    src/mosquitto.js    el broker de este equipo: usuarios (cifrado de mosquitto_passwd: $7$, PBKDF2-SHA512) que Vigía
                        escribe en <datos>/mosquitto/passwd; vigia-mosquitto.path (root) los pasa a Mosquitto y recarga
    src/homeassistant.js  salida a Home Assistant por MQTT con autodescubrimiento (los temas, arriba del fichero)
    src/paneles.js      paneles: nombres, paneles sin nodo (números negativos: -1, -2…), datos MQTT de cada panel
                        (claves «mq1», «mq2»…: lecturas, alarmas e histórico como una métrica más; una lectura por
                        minuto) y lo del receptor que sale a la malla como su telemetría; migra la estación de la 1.16
                        (lo común con la pantalla, en nucleo/src/mqtt-datos.js)
    instalar.sh sistema/  instalador (Node propio en /opt/vigia, Mosquitto por apt), servicios systemd, regla de udev
    test/nodo-falso.mjs nodo Meshtastic simulado (protobuf de verdad) por TCP o puerto virtual
    test/mqtt.test.mjs paneles.test.mjs  con un broker en memoria (aedes) y un «Home Assistant» de mentira
    test/mosquitto.test.mjs  con un Mosquitto de verdad si está instalado (apt install mosquitto)
  herramientas/         pantalla-servidor.mjs: de punta a punta con Playwright (ver herramientas/LEEME.md)
  dist/app/             app compilada (no va al repo)
```

El motor recibe el objeto de estado `S` (ver `estadoInicial()`) y ganchos; la app le añade a `S` lo de pantalla.
La app ligera congelada tiene su propia copia del código en `main`: aquí ya no hace falta mantener el formato de su
`localStorage` ni que la app sin servidor quede igual.

### Decisiones del servidor (Dani, 18 oct. 2026)

- Nodo propio por USB para el servidor (su DNN 5); Home Assistant sigue con el suyo por WiFi.
- Sin contraseña ni token: la red de casa es la protección. Puerto web 4404.
- Manda por la malla todo lo que la app: peticiones que se pulsan y avisos automáticos de alarmas (si se activan).
- Histórico de todos los nodos que oye el receptor, un año por defecto (`retencionDias`).
- Con un año de todo, nada que recorra la tabla de lecturas entera en el camino normal: el total va en memoria y el
  CSV sale por bloques.

### MQTT y Home Assistant (Dani, 18 oct. 2026)

- Un **cliente MQTT general** en el servidor (Home Assistant primero; más adelante, quizá zigbee2mqtt o Shelly como
  entradas, con un Mosquitto en la misma máquina: solo es la dirección, no está pedido).
- **No quita nada de Vigía**: sonido, notificaciones y avisos por la malla siguen igual; HA es una salida más. Si Dani
  no quiere que Vigía avise, lo silencia él en Vigía.
- Se configura en **Ajustes → MQTT** (solo en modo Servidor): los brokers (con usuario y contraseña opcionales; la
  contraseña se guarda en SQLite y nunca sale por la API) y, aparte, en cuál se publica para Home Assistant (apagado
  por defecto).
- A HA van **solo los nodos vigilados**, como dispositivos «Vigía · nombre»; **sin posición**. Un sensor binario y un
  botón «Enterado» por cada alarma puesta, «Enterado todas» y un evento por alarma/resuelta/vista para notificar.
  Las alarmas y límites se cambian solo en Vigía.

### De MQTT a la malla (Dani, 18 oct. 2026)

- Para la estación meteorológica de su barrio (en la misma ubicación que el nodo): temperatura, humedad, presión,
  lluvia, viento… salen por el DNN 5 como su telemetría de ambiente (protobuf `Telemetry`, `environmentMetrics`).
- Cada 30 min por defecto (alineado al reloj); canal y saltos a elegir (el canal puede ser público: es su decisión).
- Los valores se guardan también como **lecturas del receptor** (gráficas, alarmas, histórico), sin tocar «oído por
  última vez» (no llegan por radio). Se guardan aunque el receptor esté desconectado.
- Un dato por tipo (la telemetría no lleva nombres). Sin dirección del viento todavía: Vigía no la conoce como métrica.
- Saltos distintos de los del receptor: la librería no deja ponerlos, el paquete se arma en `servidor/src/receptor.js`.

### Paneles y dispositivos MQTT (Dani, 21 oct. 2026) — plan por fases

Vigía servidor deja de ser solo Meshtastic: los paneles muestran también dispositivos MQTT (Shelly, Tasmota, Sonoff
con Tasmota/ESPHome, Zigbee por zigbee2mqtt, ESPHome, Home Assistant…).
- **Datos MQTT dentro de los paneles**, configurados desde el propio panel (broker, tema, campo, unidad), no en
  Ajustes. Cualquier panel puede llevarlos: el de un nodo Meshtastic o uno nuevo sin nodo («Añadir panel»).
- **Solo los del panel del receptor (DNN 5)** pueden salir a la malla como telemetría (casilla por dato; uno por
  tipo). La estación de la 1.16 pasa a ser datos MQTT del panel del DNN 5 (con su histórico).
- Todos los paneles se pueden **renombrar** (los nodos guardan su nombre de Meshtastic como información). Solo en
  modo Servidor; la app de Bluetooth no cambia.
- Cada dato MQTT con nombre propio; varios del mismo tipo en un panel (salvo para la malla).
- Números y **estados ON/OFF** (alarma «cuando esté ON/OFF»), **control** con interruptor (sin confirmación; sin
  login por ahora: más adelante habrá).
- Lo de los nodos sigue yendo a Home Assistant como hasta ahora; los datos MQTT, no (si Dani los quiere en HA, los
  lee del broker de Vigía).
- Ubicación y mapa, solo nodos Meshtastic.
- **Brokers**: Mosquitto propio en el BMAX (instalador), **con usuario y contraseña**, fáciles de cambiar desde
  Vigía; Vigía usa a la vez el suyo y otros (el de HA). Cada broker se apunta una vez; en cada dato se elige cuál, y se
  puede añadir uno desde el propio panel.
- Hecho así (fase 2): un panel sin nodo es un «nodo» de número negativo en `S.nodes` (con `fuente: "mqtt"`,
  `tipoPanel`) vigilado con «sin datos» apagado; el nombre de Vigía de un nodo va en `n.alias` (el de Meshtastic sigue
  en `n.long`). Las alarmas de datos MQTT no van a Home Assistant (ni eventos, ni cuenta, ni «Enterado todas»): el
  registro las marca con `mqtt: true`. Los paneles sin nodo no salen en «Resto de la malla» ni en el mapa.
- Hecho así (fase 3, Dani 21 oct. 2026): tipo de dato «estado» (1/0) con nombres a elegir (`NOMBRES_ESTADO`) y «al
  revés»; `n.metrics[k].desde` = desde cuándo tiene ese valor. Alarma `rules[k].estado = { cuando, min }` («cuando:
  Abierta», al momento o «más de X min»; `evalEstados` cada 15 s en el servidor). Lecturas de un estado: cada cambio
  al momento, si no cambia una por hora. Tema de conexión (LWT) por panel sin nodo: `n.conectado`, alarma
  `${num}||conexion` si se marca «Avisar si se desconecta». Las alarmas de los datos MQTT siguen sin ir a Home
  Assistant: Dani valorará más adelante mandarlas a HA o a Telegram con un bot.
- Hecho así (fase 4, Dani 22 oct. 2026): un estado puede llevar `orden: { tema, on, off }` (la propone
  `proponerOrden` desde el tema de estado; se puede cambiar). Interruptor en tarjeta, ficha y pantalla completa, sin
  confirmación; `paneles.mandar` publica (qos 1, sin retener) y espera a que el tema de estado diga el valor (10 s;
  si no, «No ha contestado»); queda en el registro. A los Shelly se les pide el estado al conectar con el broker y al
  guardar los datos (`pedirEstadoShelly`: `status_update` en `…/command/switch:0`): solo información.
- Fases: 1) brokers (Mosquitto propio + varios); 2) paneles, nombres, datos MQTT numéricos, estación al panel del
  receptor; 3) ON/OFF; 4) control; 5) descubrimiento automático (**congelada** por Dani el 22 oct. 2026 hasta que
  haya más dispositivos compatibles; él tiene Shelly Plus/Pro/Gen3; idea: «Añadir panel» → «Buscar dispositivos»);
  6) **controlar interruptores desde la malla** (a Dani le interesa mucho). Más adelante: un segundo receptor 868 con
  firmware propio para estaciones meteorológicas cercanas (si publica por MQTT, es otra fuente más).

### Tailscale (Dani, 22 oct. 2026)

- Para abrir la pantalla fuera de casa: Tailscale en el BMAX (ya unido a su cuenta) y `sudo tailscale serve --bg 4404`
  (https en `*.ts.net`, instalable como app). `tailscale serve` deja el Host tal cual, así valen las comprobaciones de
  `web.js` (`ts.net` está entre los nombres de casa). Lo que llega por **Funnel** (cabecera `Tailscale-Funnel-Request`)
  se rechaza: Vigía no tiene contraseña. Más adelante: notificaciones push y quizá usar `Tailscale-User-Login` para
  saber quién manda cada orden.

### Versión 2.0 (Dani, 22 oct. 2026) — plan acordado

- **Solo el servidor**: la webapp de Bluetooth/cable se **congela** en la 1.20 y sigue publicada como versión ligera
  (solo arreglos graves); sale del repositorio principal (rama o repositorio aparte). La interfaz de la v2 es solo la
  del servidor.
- **Nombre nuevo** (sin «Mesh»: ya no es solo Meshtastic). A Dani no le gusta cómo suena «Vigía» (la g). Busca algo
  con o sin «Naika» (su marca: antes NaikaLabs en GitHub) que evoque red, vigilancia, nodo, telemetría…; le valen
  palabras en euskera. A Dani le gusta mucho **Talaia** (atalaya en euskera y en valenciano); falta una palabra
  detrás, descriptiva, para diferenciarse y por temas legales (existió Talaia Networks, de monitorización de redes,
  comprada por Auvik en 2018: mirar la EUIPO). Otras candidatas: Zaindari, Behatoki.
- **Idioma**: README y documentación en inglés (con versión en español); la app en español e inglés (elegible),
  hecho junto al rediseño; código nuevo en inglés y lo existente se renombra por partes al reescribirlo, con las
  pruebas como red. Los commits pasan al inglés cuando empiece la v2.
- **Rediseño**: barra lateral de iconos en ordenador y barra inferior en el móvil; secciones Tablero, Alarmas, Mapa,
  Automatizaciones, Ajustes (MQTT, Malla, Receptor, Sistema). Estilo de instrumento de campo, sin adornos: oliva o
  verde bosque con arena, un color de señal para las alarmas, números monoespaciados, modo oscuro «guardia nocturna».
  Logo: torre de vigilancia forestal geométrica que se lea a 16 px. Sin propuestas visuales previas (Dani se fía del
  planteamiento).
- **Usuarios y contraseña** obligatorios antes de abrirlo al mundo; **Docker** para instalarlo fácil.
- Paso 2 hecho (22 oct. 2026): `main` es la app ligera congelada (con aviso en su README), `version-1.20` guarda la
  1.20 y `mapa` ya no lleva los ficheros compilados de la app ligera ni sus herramientas. Queda para cuando haya nombre:
  crear el repositorio nuevo (lo crea Dani en GitHub) y llevar `mapa` allí con su historia. El código de la app ligera
  que sigue en `app/src` (Bluetooth, cable, demo, IndexedDB) se quita con el rediseño (paso 3).
- Orden: 1) nombre y política de idioma; 2) congelar la webapp y reorganizar el repositorio; 3) rediseño con los dos
  idiomas; 4) usuarios; 5) Docker y documentación en inglés → **2.0**. Las **automatizaciones** («si… entonces…»,
  centradas en la malla, con la fase 6 de control desde la malla dentro) → **2.1**. Nada de editor de flujos.

## Publicar

- **Desarrollo de la 2.0** en `mapa`. Dani lo prueba instalando desde la rama:
  `wget -qO- https://raw.githubusercontent.com/Dani-NRos/vigia-mesh/mapa/codigo/servidor/instalar.sh | sudo bash -s -- --rama mapa`
  Ya **no** se sube `mapa:main` (`main` es la app ligera). La 2.0 se publicará en el repositorio nuevo; hasta entonces,
  la versión estable del servidor es la 1.20 (instalador de `main`).
- **`INSTALADOR=` en `instalar.sh`**: subirlo cada vez que cambia lo que hace el instalador. Si el programa descargado
  trae uno más nuevo, el instalador que corre sigue con ese (`--programa`), pero eso solo lo hacen los de la 2 en adelante.
- **Arreglos en la app ligera**: rama desde `main`, compilar, copia en `beta/` sin `sw.js` (con `beta/README.md`),
  Dani la prueba, y se publica en `main` (subir versión 1.20.x, compilar y copiar `codigo/dist/app/*` a la raíz,
  borrar `beta/`). Las herramientas para comprobar que no cambia nada más (`capturar.mjs`, `real.mjs`,
  `comparar.py`, con Playwright y el reloj congelado) están en `main`.

## Lo aprendido con la malla y el firmware (respetarlo)

- **Acuses por id de paquete**, no fiarse de la librería: el primer acuse suele ser el implícito del propio receptor
  (oye a un vecino repetir el paquete). Entregado solo cuando confirma el nodo destino.
- **Una respuesta se reconoce por cuándo llega**, no por la hora del paquete: el reloj del receptor puede ir desfasado
  (el de Dani va unos minutos atrasado).
- Su malla: `usePreset: false`, **SF7**, límite de **6 saltos**, canal «SFNarrow». La calidad del SNR depende del SF.
- **Traceroute**: uno cada 30 s como mucho (lo limita el firmware).
- **Ficha (NodeInfo)**: el nodo no contesta a quien se la pidió en las últimas 12 h.
- **Troceado serie**: el de `@meshtastic/core` tira paquetes con 0x94 0xC3 dentro; usar `nucleo/src/tramas.js`.
  Al reconectar un puerto serie, esperar a que el anterior termine de cerrarse.
- Su receptor: **DNN 5** (nRF52, USB 239a:8029). Su base de nodos tiene unos 1100 (muchos llegados por MQTT).
- **Firmware (comprobado en 2.8.2, oct. 2026)**:
  - Por TCP (puerto 4403) solo hay un cliente: uno nuevo cierra al anterior («Force close previous TCP connection»).
  - Los paquetes recibidos van a **una sola cola compartida** entre todos los clientes (Bluetooth, cable, TCP): si hay
    dos conectados al mismo nodo, cada uno se queda con una parte. El servidor tendrá su propio nodo por USB; Home
    Assistant mantiene el suyo por WiFi.
  - Sin tráfico del cliente, el modo API se corta a los 15 min (cable y TCP): mandar latido (`tramas.js`).

## Comportamiento aceptado

- Avisos por la malla: si saltan varias alarmas en el mismo instante y van a un solo destino, salen seguidos sin los
  8 s de separación (los que se juntan en cola sí los respetan). Dani decidió dejarlo así (18 oct. 2026); el README ya
  no promete los 8 s. Lo fija una prueba en `nucleo/test/motor.test.mjs`.
