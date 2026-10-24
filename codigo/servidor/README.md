# Vigía servidor

Vigía Mesh funcionando 24/7 en Linux: las mismas alarmas que la app (umbral, cambio rápido, valla antirrobo, sin
señal) y el histórico de lecturas en SQLite, aunque no haya ningún móvil ni ordenador mirando.

> **Esta es la versión 1.20, que ya no cambia.** Vigía servidor sigue como **[Talaia Hub](https://github.com/Dani-NRos/talaia-hub)**
> (versión 2.0), con su propio instalador y su [manual](https://github.com/Dani-NRos/talaia-hub/blob/main/docs/guide.es.md).
> Si ya tienes Vigía servidor, el instalador de Talaia lo pasa a Talaia con tus datos y tu configuración:
> `wget -qO- https://raw.githubusercontent.com/Dani-NRos/talaia-hub/main/codigo/servidor/instalar.sh | sudo bash`

## Qué hace falta

- **Un ordenador con Linux**: un mini PC con Ubuntu o Debian, o una Raspberry Pi 4, 5 o Zero 2 W con Raspberry Pi OS
  de **64 bits** (en 32 bits funciona con Node 22, con soporte hasta abril de 2027). La Zero W original no vale.
- **Un nodo Meshtastic solo para el servidor**, enchufado por USB con un cable de datos, o accesible por la red (WiFi,
  Ethernet o una Raspberry con placa LoRa y `meshtasticd`).

**Un nodo, un cliente.** El firmware reparte lo que recibe entre todos los clientes conectados a la vez (Bluetooth,
cable y red): si el servidor y otro programa usan el mismo nodo, a cada uno le llega solo una parte de las lecturas.
Además, por la red solo admite un cliente: uno nuevo echa al anterior. Si Home Assistant ya usa un nodo con la
integración de Meshtastic, el servidor necesita otro.

## Instalar

Con `wget`, que viene en Ubuntu y en Raspberry Pi OS (con `curl` también vale: `curl -fsSL` en vez de `wget -qO-`):

```
wget -qO- https://raw.githubusercontent.com/Dani-NRos/vigia-mesh/main/codigo/servidor/instalar.sh | sudo bash
```

Por defecto usa el nodo que haya enchufado por USB. Para un nodo por la red:

```
wget -qO- https://raw.githubusercontent.com/Dani-NRos/vigia-mesh/main/codigo/servidor/instalar.sh | sudo bash -s -- --tcp 192.168.1.60
```

El instalador:

- pone su propio Node en `/opt/vigia/node`, sin tocar el del sistema;
- deja el programa en `/opt/vigia/actual` y guarda las dos versiones anteriores;
- crea un usuario `vigia` sin privilegios, con permiso para el puerto serie (grupo `dialout`);
- escribe la configuración en `/etc/vigia/servidor.json` y guarda los datos en `/var/lib/vigia`;
- instala **Mosquitto** (el broker MQTT de casa) en el puerto 1883, **con usuario y contraseña**; los usuarios se
  crean desde la pantalla (ver *MQTT* más abajo). Si Mosquitto ya tenía su propia configuración, no la toca. Para no
  instalarlo: `--sin-mosquitto`;
- deja el servicio `vigia-servidor` en marcha, y arranca solo al encender.

Para **actualizar**, vuelve a ejecutar la misma orden: se queda con tu configuración y tus datos.

## Seguirlo

```
journalctl -u vigia-servidor -f       lo que va pasando
vigia-servidor --puertos              qué nodos hay enchufados por USB
systemctl status vigia-servidor       si está en marcha
```

En el diario salen la conexión con el receptor, las alarmas, las resueltas, los fallos (con qué hacer) y un resumen
cada hora.

## Configuración

`/etc/vigia/servidor.json`; tras cambiarla, `sudo systemctl restart vigia-servidor`.

```json
{
  "receptor": { "tipo": "usb", "puerto": "auto" },
  "web": { "puerto": 4404 },
  "datos": "/var/lib/vigia",
  "retencionDias": 365,
  "registro": "info"
}
```

- `receptor.puerto` por USB: `"auto"` (el nodo que haya enchufado), un id USB como `"239a:8029"` (sigue valiendo si
  el puerto cambia de nombre) o una ruta fija, mejor de `/dev/serial/by-id/` (el nombre de `vigia-servidor --puertos`).
- Por la red: `{ "tipo": "tcp", "host": "192.168.1.60", "puerto": 4403 }`.
- `retencionDias`: cuánto histórico se guarda (por defecto, un año). Guarda la telemetría de **todos** los nodos que
  oye el receptor, y cada lectura ocupa unos 80 bytes: 50 nodos con 4 sensores cada media hora son unos 22 MB al mes
  (unos 270 MB al año). Si tu receptor oye cientos de nodos (por ejemplo, llegados por MQTT), pueden ser varios GB al
  año: en una Raspberry con tarjeta SD, baja los días. El diario dice cada 6 horas cuántas lecturas hay y cuánto ocupa.
- `registro`: `"depurar"` cuenta también cada reintento de conexión.

## Si no conecta

El diario dice qué pasa y qué hacer. Lo más habitual:

- **No hay ningún nodo enchufado**: un cable que solo carga, o el nodo apagado. Al enchufarlo conecta solo en menos de
  un minuto.
- **El puerto está ocupado**: otro programa lo tiene abierto (la web de Meshtastic, un monitor serie…). En Ubuntu de
  escritorio, `brltty` se queda con los nodos de chip CH340: si no usas una línea braille, `sudo apt remove brltty`.
- **Por la red, se corta a menudo**: otro cliente (la app de Meshtastic, Home Assistant…) se está conectando al mismo
  nodo y echa al servidor.

## Desinstalar

```
wget -qO- https://raw.githubusercontent.com/Dani-NRos/vigia-mesh/main/codigo/servidor/instalar.sh | sudo bash -s -- --desinstalar
```

Los datos se quedan en `/var/lib/vigia`; con `--desinstalar --borrar-datos` se borran también. Mosquitto se queda
instalado, sin la configuración de Vigía (si no lo usas para nada más: `sudo apt remove mosquitto`).

## La pantalla

Al terminar, el instalador dice en qué direcciones abrirla, como `http://192.168.1.50:4404`. Es la misma app de
Vigía, pero los datos y el receptor son los del servidor: todos tus aparatos ven lo mismo, y si das «Enterado» en uno,
queda visto en todos. El sonido y la pantalla encendida son de cada aparato.

- **Sin contraseña**: la protege la red de casa. Cualquiera conectado a tu WiFi puede abrirla, ver las posiciones de
  los nodos y mandar mensajes por la malla con tu receptor.
- **Por http, el navegador no deja** instalarla como app ni mostrar notificaciones con la página cerrada. Eso llega
  con https (Tailscale). Mientras tanto, la página suena y avisa mientras está abierta.
- **No conectes también el móvil por Bluetooth** al nodo del servidor: el nodo repartiría lo que recibe entre los dos.

## Desde fuera de casa (Tailscale)

Con [Tailscale](https://tailscale.com), la pantalla se abre desde el móvil o el portátil fuera de casa, sin abrir
puertos en el router: solo los aparatos de tu red de Tailscale llegan al servidor.

1. **El servidor en Tailscale**: instalado y unido a tu cuenta (`tailscale status` lo dice, con su nombre y su IP
   `100.…`). Así ya se puede abrir `http://100.x.y.z:4404` (o `http://nombre-del-equipo:4404`) desde cualquier aparato
   de tu red de Tailscale.
2. **Con https (recomendado)**: en la consola de Tailscale, *DNS*, deja activado *MagicDNS* y pulsa *Enable HTTPS*
   (en *HTTPS Certificates*). En el servidor:
   ```
   sudo tailscale serve --bg 4404
   ```
   Dice la dirección, como `https://bmax.tail1234.ts.net`. Se queda puesto aunque se reinicie el equipo
   (`tailscale serve status` para verlo; `sudo tailscale serve --https=443 off` para quitarlo). La primera vez tarda
   unos segundos: está sacando el certificado.
3. **En el móvil**: la app de Tailscale con la misma cuenta, conectada, y abre esa dirección en Chrome. Con https se
   puede **instalar como app** (menú de Chrome → *Instalar aplicación*) y dejar que avise con notificaciones mientras
   está abierta. Es otra app distinta de la de GitHub (la del servidor, con sus datos).

Lo que hay que saber:

- **Nada de Tailscale Funnel**: Funnel publica en internet, y Vigía no tiene contraseña. Si llega algo por Funnel,
  Vigía lo rechaza y lo apunta en el diario. Usa `tailscale serve`, que solo es para tu red de Tailscale.
- Cualquiera que esté en tu red de Tailscale puede abrir la pantalla y manejarla (mandar por la malla, encender y
  apagar). Si compartes tu red de Tailscale con alguien, tenlo en cuenta.
- El certificado de https sale en los registros públicos de certificados: el nombre del equipo y de tu red de
  Tailscale se pueden ver (no se puede entrar con ellos).
- Las notificaciones con la app cerrada (push) llegarán más adelante.

## Avisos por la malla

Los activas en Ajustes → *Avisos por la malla*, como en la app (vienen apagados). Una vez activados, los manda el
servidor 24/7, aunque no haya ninguna pantalla abierta: cada alarma (y, si quieres, cuando se resuelve) a un canal
privado, a un nodo o a ambos. Como mucho 20 por hora; los canales públicos están bloqueados.

## MQTT

Vigía se conecta a la vez a varios brokers MQTT, que se apuntan en Ajustes → *MQTT*:

- **Este servidor**: el Mosquitto de este equipo, el que pone el instalador. Ahí pueden publicar los dispositivos de
  casa (Shelly, Tasmota, zigbee2mqtt, ESPHome…). Solo deja entrar con usuario y contraseña: en *Usuarios de este
  broker* se crean (por ejemplo, uno para los Shelly), se les cambia la contraseña (guardando otra vez el mismo
  usuario) y se quitan. En el dispositivo: servidor, la IP de este equipo; puerto, 1883; y ese usuario. Las
  contraseñas se guardan cifradas y no vuelven a salir en ninguna pantalla.
- **Otros brokers** con *Añadir broker*, como el Mosquitto de Home Assistant: nombre, dirección, usuario y
  contraseña. Debajo de cada uno dice si está conectado o qué falla.

Por dentro, Vigía escribe los usuarios en `/var/lib/vigia/mosquitto/passwd` y el servicio `vigia-mosquitto.path` los
pasa a Mosquitto y lo recarga, sin cortar a quien ya está conectado. Si usas un cortafuegos (`ufw`), abre el 1883 para
la red de casa.

Home Assistant solo se conecta a un broker. Si quieres en Home Assistant algo que se publica en el de Vigía, lo normal
es un «puente» entre los dos Mosquitto (unas líneas de configuración en uno de ellos).

## Home Assistant

Home Assistant crea solos los dispositivos de Vigía. En Ajustes → *MQTT* → *Home Assistant*: activa *Publicar los
nodos vigilados*, elige el broker de Home Assistant y pulsa *Guardar* (viene apagado).

**Con el complemento Mosquitto de Home Assistant:**

1. En Home Assistant, crea un usuario para Vigía: Ajustes → Personas → *Añadir persona* (por ejemplo `vigia`), con
   permiso para iniciar sesión, y ponle contraseña. Mosquitto deja entrar con los usuarios de Home Assistant (los
   nombres `homeassistant` y `addons` están reservados).
2. En la pantalla de Vigía servidor, Ajustes → *MQTT* → *Añadir broker*: nombre (Home Assistant), la dirección de
   Home Assistant (como `192.168.1.20`; si no pones puerto, usa el 1883), ese usuario y su contraseña. Luego, en
   *Home Assistant*, actívalo y elige ese broker.
3. En Home Assistant aparecen solos, en Ajustes → Dispositivos y servicios → MQTT:
   - **Vigía servidor**: receptor conectado, alarmas sin atender (sí/no y cuántas), alarmas activas, nodos oídos en la
     última hora, último aviso por la malla, el evento *Alarmas* y el botón *Enterado todas*.
   - **Vigía · Caseta huerto** (uno por cada nodo vigilado): sus lecturas (temperatura, humedad, batería…), cuándo se
     oyó por última vez, saltos y SNR; y por cada alarma que le tengas puesta, un sensor (encendido mientras salta,
     con el detalle en sus atributos) y su botón *Enterado*. La posición no se manda.

Las alarmas y los límites se siguen poniendo en Vigía: Home Assistant los ve y puede dar «Enterado», pero no los
cambia. Va solo lo de los nodos Meshtastic: los datos MQTT de los paneles y sus alarmas, no (ni cuentan en las
alarmas activas, ni *Enterado todas* las da por vistas). Si renombras un nodo en Vigía, su dispositivo cambia de
nombre. Si dejas de vigilar un nodo, su dispositivo desaparece de Home Assistant. Si el servidor se apaga o se cae,
Home Assistant marca sus entidades como no disponibles. Vigía sigue sonando y avisando como siempre: Home Assistant es
una salida más.

**Avisar al móvil con la app de Home Assistant.** Cada alarma que salta, se resuelve o se da por vista llega como
evento (`event.vigia_servidor_alarmas`, con `event_type` *alarma*, *resuelta* o *vista*, y en sus atributos `texto`,
`detalle`, `nodo` e `id`). Una automatización (en YAML; cambia `mobile_app_mi_movil` por tu móvil, y comprueba el
nombre de la entidad *Alarmas* en el dispositivo *Vigía servidor*):

```yaml
alias: Vigía → móvil
triggers:
  - trigger: state
    entity_id: event.vigia_servidor_alarmas
    not_from: [unavailable, unknown]   # que no avise al reiniciar Home Assistant o el servidor
conditions:
  - condition: template
    value_template: "{{ trigger.to_state.attributes.event_type == 'alarma' }}"
actions:
  - action: notify.mobile_app_mi_movil
    data:
      title: Vigía
      message: "{{ trigger.to_state.attributes.texto }}. {{ trigger.to_state.attributes.detalle or '' }}"
```

**Temas** (por si los usas desde otro programa; el tema base se cambia en Ajustes → *Temas*):

```
vigia/estado                     online / offline (retenido; offline también si el servidor se cae)
vigia/servidor/estado            JSON: receptor, sin_atender, activas, oidos, aviso (retenido)
vigia/servidor/evento            JSON por cada alarma, resuelta o vista (no retenido)
vigia/servidor/enterado_todas    ← cualquier mensaje: «Enterado» a todas
vigia/nodo/<id>/estado           JSON: m (lecturas), oido, saltos, snr, a (alarmas ON/OFF), d (su detalle) (retenido)
vigia/nodo/<id>/enterado         ← el nombre de la alarma (como temperature_alta): «Enterado» a esa
homeassistant/…/vigia_…/config   autodescubrimiento de Home Assistant (retenido)
```

`<id>` es el número del nodo en hexadecimal, sin `!` (como `3c6e19a2`).

**Sin contraseña en la pantalla**: cualquiera en tu WiFi puede cambiar a qué broker se conecta el servidor o apagar
MQTT (no ver la contraseña guardada).

## Paneles y datos por MQTT

Con el servidor, el tablero no es solo de nodos Meshtastic: cualquier dispositivo que publique por MQTT (Shelly,
Tasmota, Sonoff, Zigbee con zigbee2mqtt, ESPHome, Home Assistant…) puede tener su panel, con sus lecturas, gráficas,
alarmas (también avisos por la malla) e histórico, como un nodo. Solo en el servidor: la app por Bluetooth no cambia.

- **Panel nuevo**: *Añadir panel*, encima de los sensores vigilados. Un nombre y qué es (solo para la etiqueta).
- **Datos por MQTT**: en la ficha de cada panel (también en la de un nodo, para tener junto a él otro dato, como la
  temperatura de dentro de la caseta). Cada dato lleva su nombre (puede haber varias temperaturas), qué es, de qué
  broker viene, el tema y, si llega un JSON, el campo (`temperature`, o `ENERGY.Power` si va dentro de otro). Si el
  viento llega en km/h o la temperatura en °F, se elige la unidad y Vigía la pasa a la suya. Debajo de cada dato se
  ve el último valor que ha llegado. *Otro número* sirve para lo que no está en la lista (con la unidad que pongas).
- **Buscar temas** escucha los brokers unos segundos y enseña los temas con números (se pueden filtrar). **Usar**
  pone uno en la lista: adivina qué es por el nombre (temperature, humidity, Power…) y, si Home Assistant publica la
  unidad, la elige; si no, el viento queda en km/h, lo normal en Home Assistant. Compruébalo.
- **Lecturas**: el valor se ve y salta la alarma al momento; en el histórico se guarda uno por minuto como mucho.
- **Estados ON/OFF** (*Estado ON/OFF* en el tipo): un enchufe, un relé, una puerta, un detector de agua… Vigía
  entiende `ON`/`OFF`, `on`/`off`, `true`/`false`, `1`/`0`, `open`/`closed`, `online`/`offline` y un JSON con
  cualquiera de ellos (`{"POWER":"ON"}` de Tasmota, `{"output":true}` de Shelly, `{"contact":false}` de
  zigbee2mqtt). Se eligen sus nombres (*Encendido / Apagado*, *Abierta / Cerrada*, *Hay agua / Seco*…) y, si el
  dispositivo lo dice al revés, *Al revés*: el `contact` de zigbee2mqtt es `true` con la puerta **cerrada** (al usarlo
  desde *Buscar temas* ya se marca). Cada cambio se guarda al momento en el histórico (si no cambia, uno por hora) y
  su gráfica es una barra de tiempo con lo que estuvo encendido.
- **Shelly de segunda y tercera generación** (Plus, Pro, Gen3): en el Shelly, *Ajustes → MQTT*, activa *Generic status
  update over MQTT*; el estado completo sale en `<prefijo>/status/switch:0` (campos `output`, `apower`, `voltage`,
  `aenergy.total` en Wh, `temperature.tC`), y la conexión en `<prefijo>/online`. Sin esa opción, solo está
  `<prefijo>/events/rpc`, con el campo `params.switch:0.output`: llega solo cuando algo cambia (*Buscar temas* lo ve si
  pulsas el enchufe mientras busca) y a veces sin ese campo (solo la potencia): esos mensajes no cuentan para el dato.
- **Alarma de un estado**, en la ficha: *Avisar cuando: Abierta* (o *Cerrada*), al momento o solo si sigue así más de
  unos minutos (una puerta abierta más de 10 min, una bomba encendida más de 1 h). Se resuelve al cambiar.
- **Encender y apagar desde Vigía**: en un estado, *Encender y apagar desde Vigía* y el tema al que se mandan las
  órdenes, con el mensaje para cada estado. Vigía lo propone a partir del tema de estado (Shelly Plus/Pro/Gen3:
  `…/command/switch:0` con `on` y `off`, con *Enable MQTT Control* activado en el Shelly; Tasmota: `cmnd/…/POWER` con
  `ON` y `OFF`; zigbee2mqtt: `zigbee2mqtt/…/set` con `{"state":"ON"}`; ESPHome: `…/switch/…/command`); se puede
  cambiar. En la tarjeta, la ficha y la pantalla completa sale un interruptor: se pulsa y se manda, **sin
  confirmación** y sin contraseña (cualquiera en la red de casa puede usarlo). Dice *Enviando…* hasta que el
  dispositivo cuenta por su tema de estado que lo ha hecho; si en 10 s no lo cuenta, *No ha contestado*. Cada orden
  queda en el registro. Home Assistant no recibe órdenes por `mqtt_statestream`.
- **Los Shelly no guardan su estado en el broker**: al conectar con él (y al guardar los datos), Vigía les pide que lo
  digan (`status_update` en `…/command/switch:0`; en los de primera generación, `update`). Solo es una petición de
  información, no cambia nada. Por `…/events/rpc` no se puede pedir: mejor `…/status/switch:0`.
- **Conexión del dispositivo** (paneles sin nodo, opcional): el tema donde dice si está conectado. Tasmota:
  `tele/…/LWT` (`Online`/`Offline`); Shelly: `…/online` (`true`/`false`); zigbee2mqtt: `zigbee2mqtt/…/availability`
  (campo `state`). Con él, el panel dice *Sin conexión* al momento y, con *Avisar si se desconecta*, salta su alarma
  (y su aviso por la malla, si están activados). *Buscar temas* lo reconoce y ofrece *Usar como conexión*.
- **Nombre**: todos los paneles se pueden renombrar. El de un nodo es solo para Vigía (tablero, alarmas, avisos por
  la malla y Home Assistant): su nombre de Meshtastic se queda como información y, con el nombre vacío, vuelve a él.
- **Sin datos**: en un panel sin nodo, la alarma «no llega ningún dato en más de…» viene apagada, porque hay
  dispositivos que solo publican cuando algo cambia.
- **Quitar panel** (al final de su ficha) lo borra con su histórico. Al quitar un dato, se borra también el suyo.

La ubicación y el mapa siguen siendo solo de los nodos Meshtastic. A Home Assistant no van los datos MQTT ni sus
alarmas: si los quieres allí, Home Assistant puede leerlos del broker donde se publican. (El sensor *último aviso por
la malla* sí enseña el texto del último mensaje que salió por la malla, sea de lo que sea.)

## A la malla (el panel del receptor)

Los datos MQTT del panel del receptor pueden salir a la malla cada cierto tiempo como **su telemetría de ambiente**,
como si tuviera esos sensores: la app de Meshtastic y Vigía los ven en sus gráficas. Es lo de una estación del barrio
en la misma ubicación que el nodo.

- En cada dato del receptor, la casilla **A la malla**, y debajo de *Datos por MQTT*, **Mandar a la malla**: **cada**
  cuánto (por defecto, 30 minutos, a las en punto y a las y media), **por qué canal** y con **cuántos saltos** (por
  defecto, los del receptor). Viene apagado.
- **Enviar ahora** manda lo que haya sin esperar (como mucho, uno por minuto).

Lo que hay que saber:

- **Un dato de cada tipo.** La telemetría de Meshtastic no lleva nombres: tiene un hueco por tipo de dato. Puede
  haber varias temperaturas en el panel del receptor, pero solo una sale a la malla. Potencia, energía o CO₂ no tienen
  hueco en la telemetría de ambiente: se ven en el panel, pero no salen.
- **Salen como medidos por el receptor y en su posición.** Para una estación del barrio, bien; para datos de otro
  sitio, quien los vea en el mapa los pondrá donde está el receptor.
- **Si el receptor tiene sensores de ambiente propios**, sus valores se mezclarían con los de la estación.
- **«No disponible» no se manda.** Si Home Assistant dice `unavailable` o `unknown`, ese dato no sale. Como Home
  Assistant solo publica cuando el valor cambia (la lluvia puede estar a 0 días), por defecto se manda el último que
  llegó; en cada dato se puede elegir dejar de mandarlo si lleva 1, 3, 12 o 24 h sin llegar nada nuevo.
- **Canal público o privado.** Por un canal público lo ve cualquiera y puede acabar en los mapas de internet; para
  una estación del barrio puede tener sentido, para datos de casa, mejor uno privado. El firmware manda su propia
  telemetría por el canal principal; por otro canal la ven quienes lo tengan.
- **Lluvia 24 h** es «las últimas 24 horas»: si tu estación da «lluvia de hoy», saldrá con ese nombre.
- En el diario sale cada fallo (una vez, no cada media hora) y cuando vuelve a salir bien.

**Desde la 1.16**: la estación que estaba en Ajustes pasa sola al panel del receptor (que queda vigilado), con su
histórico, su canal y sus saltos. Esos datos ya no van a Home Assistant como lecturas del receptor.

**Desde Home Assistant**, sus sensores no salen por MQTT por su cuenta. Con `mqtt_statestream` en
`configuration.yaml` (y reiniciar Home Assistant):

```yaml
mqtt_statestream:
  base_topic: ha_estados
  publish_attributes: true
  include:
    entities:
      - sensor.estacion_temperatura
      - sensor.estacion_humedad
      - sensor.estacion_presion
      - sensor.estacion_viento
```

Con `publish_attributes: true` sale también la unidad de cada uno y Vigía la elige sola. Cada uno sale en
`ha_estados/sensor/estacion_temperatura/state` (retenido: Vigía lo tiene nada más conectar). No incluyas los sensores
«Vigía · …»: Vigía rechaza sus propios temas para no volver a recibir lo que ya publica él.
