# Vigía Mesh

Panel de telemetría y alarmas para nodos [Meshtastic](https://meshtastic.org) que funciona **sin internet**. El móvil se conecta por Bluetooth a un nodo receptor (en el ordenador, también por cable USB), y los sensores le llegan por LoRa.

**Abrir la app:** https://dani-nros.github.io/vigia-mesh/

> **Versión ligera, congelada en la 1.20.** Esta app (Bluetooth y cable, sin instalar nada) sigue publicada y
> funcionando, pero ya no tendrá funciones nuevas: solo arreglos de fallos graves. El desarrollo continúa en
> **Vigía servidor** (la vigilancia 24/7 en un ordenador con Linux), que tendrá nombre y repositorio propios en su
> versión 2.0. Si algo falla, [abre un aviso](https://github.com/Dani-NRos/vigia-mesh/issues).

<p align="center"><img src="capturas/panel.png" alt="Tres pantallas de Vigía Mesh en modo demostración: el panel con una alarma de temperatura activa, la ficha de un nodo con los botones para pedir datos, su ubicación y la valla antirrobo, y el mapa de sensores con la temperatura de cada nodo sobre el mapa de la Ribera y l'Horta"></p>

## Qué hace

- **Panel de sensores.** Muestra las últimas lecturas de cada nodo vigilado y una mini gráfica de 24 h. Cada tarjeta se contrae a una línea (nombre, cuándo se le oyó, batería y estado) y se pueden ordenar a tu gusto con «Ordenar», arrastrándolas o con flechas; la app lo recuerda.
- **Un nodo a pantalla completa.** Sus lecturas en grande, sus alarmas y la gráfica de cada sensor (24 h, 7 o 30 días), en vivo y con la pantalla encendida mientras lo miras. Pensado para una tableta o un monitor fijo.
- **Alarmas en la propia app, con sonido, vibración y notificación:**
  - un valor sale del mínimo o máximo que fijas por métrica;
  - un valor cambia rápido, aunque no llegue a ningún límite: por ejemplo, la presión baja 3 hPa en 3 h (suele anunciar tormenta), la batería cae un 15 % en 6 h o la temperatura sube 5 °C en una hora. Tú eliges métrica, si baja o sube, cuánto y en cuántas horas (hasta 24). Cuenta desde el valor más alto (o más bajo) de ese intervalo, necesita lecturas de al menos la mitad del intervalo para decidir y se cierra cuando el cambio baja del 70 % del límite;
  - un nodo lleva más de X minutos sin dar señal;
  - el receptor se desconecta (la app reintenta la conexión sola).
- **Pedir datos ahora.** Pide a un nodo una lectura en el momento: ambiente, batería, calidad del aire o energía (INA). Si el nodo no tiene ese sensor o no le llega la petición, la app lo dice.
- **Pedir ficha del nodo.** Desde el firmware 2.5, el receptor solo manda peticiones a nodos de los que tiene la clave pública (si no, da el error 39). Este botón pide al nodo su ficha (nombre y clave) y de paso le envía la del receptor. Un nodo no contesta a la misma petición de ficha en 12 h (y cada petición nueva vuelve a empezar la cuenta), ni si acaba de anunciar la suya; aun así, si la petición le llega, ya se queda con la clave del receptor. Si se la pides otra vez antes de 12 h, la app te avisa.
- **Buscador en «Resto de la malla».** Busca por nombre (sin importar tildes ni mayúsculas), nombre corto, ID (`!abcd1234`) o modelo, con varias palabras a la vez; con texto salen también los nodos vigilados. Filtros de un toque (con sensores, con ubicación, cerca, últimas 24 h) y orden por los oídos hace menos, saltos, distancia a tu receptor, señal o nombre. Intro abre el primero; en el ordenador, la tecla «/» lleva al buscador.
- **Ruta por la malla (traceroute).** Desde la ficha de cualquier nodo, «Trazar ruta» muestra por qué nodos pasa una petición tuya hasta él y por cuáles vuelve su respuesta, con el SNR de cada salto en verde, ámbar o rojo según el margen que deja tu preset (LongFast, MediumFast o el SF que uses). Los nodos que repiten sin la clave del canal salen sin nombre, se señala el salto más flojo y se guarda la última ruta de cada nodo. «Ver la ruta en el mapa» la dibuja con el SNR de cada tramo, y la vuelta aparte si va por otro camino. El firmware solo deja lanzar un traceroute cada 30 s.
- **Ubicación y valla antirrobo.** Muestra las coordenadas de cada nodo (se copian con un toque, se ven en el mapa de la app aunque no haya internet o se abren en Google Maps), su precisión y si vienen del GPS o están fijadas a mano. Puedes pedirle la posición al momento. Para un nodo vigilado, fijas su sitio y un radio: si dos posiciones seguidas caen fuera, salta la alarma «Fuera de su sitio». Ojo: en el canal público la posición viaja recortada a unos 730 m, y un nodo con la posición fijada a mano no delata un robo (para eso está la alarma de «sin señal»).
- **Mapa de sensores.** El botón del mapa abre a pantalla completa los nodos con ubicación y la última lectura del sensor que elijas (el selector dice cuántos nodos lo tienen), coloreada por valor. Las lecturas antiguas se ven más pálidas y un borde discontinuo avisa de que el nodo se ha movido desde que tomó ese dato. Los nodos que caen juntos se apartan para no taparse. Los nodos y las lecturas salen siempre, con o sin internet. El fondo es de OpenStreetMap y se guarda en el móvil de las zonas que ves con conexión (hasta unas 2.000 teselas, unos 40 MB): antes de ir a un sitio sin cobertura, recórrelo en el mapa con internet.
- **Avisos por la malla.** Cada alarma (umbral, cambio rápido, sin señal, fuera de su sitio) puede salir como mensaje Meshtastic a un canal privado, a un nodo o a ambos, y si quieres también cuando se resuelve. En directo a un nodo, solo cuenta como entregado cuando ese nodo lo confirma: que un vecino lo repita no basta, y si no confirma en 90 s, la app lo dice. Los canales públicos (sin cifrar, con la clave por defecto o la de eventos) están bloqueados. Como mucho 20 por hora, y los que se juntan en cola salen con 8 s de separación.
- **Histórico en el propio móvil.** Gráficas de 24 h y 7 días, y exportación a CSV.
- **Modo demostración** con nodos inventados, para probarla sin hardware.

Todos los datos se quedan en el dispositivo. La app no envía nada a ningún servidor.

## Requisitos

- **Navegador.** Chrome en Android, o Chrome/Edge en el ordenador. En iPhone no funciona porque Safari no tiene Bluetooth web.
- **Receptor.** Un nodo Meshtastic en el mismo canal que los sensores, con Bluetooth activado o, en el ordenador, enchufado por USB con un cable de datos.
- **Un solo cliente por receptor.** Por Bluetooth, ningún otro móvil puede estar conectado al receptor a la vez. Por cable, ningún otro programa puede tener abierto su puerto (la web de Meshtastic, un monitor serie…). Y aunque el nodo deje conectar a la vez un móvil por Bluetooth y un ordenador por cable (o Home Assistant por WiFi), no lo hagas: el nodo reparte lo que recibe entre los que están conectados, y a cada uno le llega solo una parte de las lecturas. Si necesitas dos clientes, usa dos nodos.

## Instalación

1. Abre la dirección de la app en Chrome.
2. Pulsa el menú ⋮ y luego **Instalar aplicación**.
3. Desde entonces abre sin internet. Para actualizarla, ábrela una vez con conexión.

## Uso

1. **Conecta el receptor.** Pulsa **Conectar receptor** y elige tu nodo. En el ordenador puedes elegir **Por cable USB**: Chrome muestra los puertos de los nodos enchufados. Con cable no hay emparejamientos ni cortes, y se reconecta sola al abrir la app y al volver a enchufarlo. Si tu nodo no sale en la lista, Ajustes te deja ver todos los aparatos Bluetooth o todos los puertos serie.
2. **Elige qué vigilar.** En *Resto de la malla*, pulsa **Vigilar** en un nodo para fijarlo arriba.
3. **Pon los límites.** Toca la tarjeta para fijar mínimo y máximo por métrica, los avisos de cambio rápido y los minutos sin señal que disparan la alarma.
4. **Para usarlo como panel fijo,** activa Ajustes → *Mantener la pantalla encendida* y deja el móvil enchufado.

## Si no conecta

La app reconoce el tipo de fallo y te dice qué hacer. Lo más habitual:

- **La app de Meshtastic tiene el nodo ocupado.** En Android sigue conectada en segundo plano aunque no la abras, y el nodo solo admite un cliente Bluetooth. Ciérrala del todo o desconecta el nodo desde ella.
- **Emparejamiento roto.** Si conecta y se corta enseguida, olvida el nodo en los ajustes de Bluetooth del móvil y vuelve a conectar; Android pedirá el PIN que sale en la pantalla del nodo (sin pantalla suele ser 123456). Pasa al actualizar el firmware o al emparejar el nodo con varios aparatos. Para un receptor fijo en casa, el modo «Sin PIN» lo evita del todo, a cambio de que cualquiera a pocos metros pueda conectarse a él.
- **Otra ventana de Vigía.** Solo una ventana puede tener el receptor; la otra ofrece «Conectar aquí».
- **Por cable: el puerto está ocupado o no contesta.** Cierra cualquier otro programa que use el puerto (web de Meshtastic, flasheador, monitor serie de Arduino o PlatformIO). Si el nodo se reinicia al enchufarlo, la app vuelve a pedirle los datos sola. Algunos cables solo cargan: usa uno de datos.

En Ajustes → *Diagnóstico de conexión* queda apuntado cada paso. Si falla, cópialo: dice en qué paso se corta.

## Avisar de un fallo

Los fallos y las sugerencias se recogen en los [avisos de GitHub](https://github.com/Dani-NRos/vigia-mesh/issues). Si es un problema de conexión, el botón **Avisar del fallo en GitHub** (en Ajustes → *Diagnóstico de conexión*) abre un aviso nuevo con el diagnóstico ya pegado. Añade el modelo del nodo receptor y su versión de firmware. Los avisos son públicos: revisa el texto antes de enviarlo.

## Límites conocidos

- **Solo en primer plano.** Las alarmas saltan con la app abierta y en pantalla; con el móvil bloqueado, Chrome congela la página. Al volver, la app recoge lo que guardó el receptor y marca como «recibida al reconectar» lo que pasó mientras tanto. Para vigilar las 24 horas está [Vigía servidor](#vigía-servidor-en-pruebas).
- **No es tiempo real.** La telemetría sale cada 30 min por defecto, así que una alarma de umbral puede llegar con ese retraso. En 868 MHz el tiempo de emisión está limitado: no conviene bajar mucho el intervalo.
- **Nodos dormidos.** Un nodo en modo de ahorro de energía no escucha mientras duerme, así que no contesta a *Pedir datos ahora*.

## Vigía servidor (en pruebas)

La misma vigilancia funcionando las 24 horas en un ordenador con Linux (un mini PC o una Raspberry Pi) con un nodo
enchufado por USB o accesible por la red: alarmas aunque no haya ningún móvil mirando, histórico en el propio
servidor, avisos por la malla y conexión con Home Assistant por MQTT (los nodos vigilados aparecen como dispositivos,
con sus lecturas, sus alarmas y «Enterado» desde Home Assistant). Sus paneles pueden mostrar también dispositivos
que publican por MQTT (Shelly, Tasmota, Zigbee…), con sus valores o estados (encendido, abierta…), alarmas e
histórico, y encenderlos y apagarlos desde la pantalla; y mandar a la malla, como telemetría
de su nodo, los datos de una estación meteorológica que lleguen por MQTT. Se abre desde el navegador en la red de casa
(o desde fuera, con Tailscale), con esta misma pantalla, y todos tus aparatos ven lo mismo. Instalación y detalles en
[`codigo/servidor`](codigo/servidor/README.md).

## Código fuente

El código está en la carpeta [`codigo/`](codigo/), en tres partes:

- [`codigo/nucleo/`](codigo/nucleo/): la lógica sin pantalla (alarmas, peticiones a nodos, rutas, avisos por la malla, conexión con el nodo). La comparten la app y el servidor.
- [`codigo/app/`](codigo/app/): la app web (pantalla, Bluetooth, cable USB, mapa y demo).
- [`codigo/servidor/`](codigo/servidor/): Vigía servidor (Node.js, SQLite, MQTT, instalador para Linux).

Para compilar y probar:

```
cd codigo
npm install
npm run build
npm test
```

La app compilada queda en `codigo/dist/app/`.

## Licencia

GPL-3.0. Vigía Mesh usa la librería oficial [`@meshtastic/core`](https://www.npmjs.com/package/@meshtastic/core), publicada con licencia GPL-3.0, así que todo el proyecto se distribuye con la misma licencia. La conexión Bluetooth es propia (`codigo/src/ble.js`): serializa las operaciones porque Android solo admite una a la vez. El texto completo está en [LICENSE](LICENSE).

El mapa usa [Leaflet](https://leafletjs.com) 1.9.4, que va incluido dentro de la app, y el fondo es de [OpenStreetMap](https://www.openstreetmap.org/copyright) (© colaboradores de OpenStreetMap, datos con licencia ODbL).

<details><summary>Licencia de Leaflet (BSD-2-Clause)</summary>

```
BSD 2-Clause License

Copyright (c) 2010-2023, Volodymyr Agafonkin
Copyright (c) 2010-2011, CloudMade
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

</details>

Meshtastic® es una marca registrada de Meshtastic LLC. Este proyecto es independiente y no está afiliado a Meshtastic.
