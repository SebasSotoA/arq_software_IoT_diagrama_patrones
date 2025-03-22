# Actividad de Plataforma IoT con Dispositivos de Múltiples Fabricantes

La plataforma busca integrar múltiples dispositivos IoT (luces, termostatos, etc.) de distintos fabricantes (Philips, Nest, etc.), de manera flexible, escalable y sin acoplamientos directos. Para ello, se utilizan tres patrones de diseño: **Bridge**, **Adapter** y **Observer**.

## Patrón Bridge

Muchos dispositivos usan protocolos de comunicación diferentes. Por ejemplo, una luz Philips puede usar un protocolo Zigbee, y un termostato Nest puede usar Wi-Fi.  
El patrón **Bridge** permite separar la forma de comunicarse (el protocolo) del comportamiento del dispositivo (lo que hace).
Así, podemos tener una luz o un termostato que no necesita saber cómo se comunica internamente, solo envía comandos a través de un “puente”.

#### ¿Qué Resuelve?

Permite desacoplar la abstracción del dispositivo (como una luz o un termostato) de la **forma en que se comunica** (protocolo del fabricante), de modo que puedan evolucionar de manera independiente.
Tiene dos jerarquías fundamentales:

##### Abstracción

`DeviceCommunication` (abstracta): representa cómo se comunican los dispositivos en general. Esta clase es **la parte del puente** que conecta la lógica del dispositivo con su protocolo real.
Tiene un atributo interno llamado `_implementation` que representa **el protocolo que se va a usar**. Cuando el dispositivo quiere enviar un comando, no habla directamente con Philips o Nest, **sino con `DeviceCommunication`**, que se encarga de traducir todo.

- Métodos: `Initialize()`, `Send(command)`, `Receive()`.
- Tiene un atributo `_implementation` del tipo `IDeviceProtocolImplementation`.

##### Implementación

`IDeviceProtocolImplementation`: interfaz que define cómo deben comportarse los protocolos (conectar, enviar comandos, recibir datos). Con los métodos `Connect()`, `SendCommand(command)`, `ReceiveData()`.

##### Implementaciones Concretas:

    - `PhilipsProtocol`
    - `NestProtocol`

Son **implementaciones concretas** de la interfaz. Cada una tiene su propia lógica para conectarse, enviar y recibir datos. Por ejemplo:

- `PhilipsProtocol` podría traducir `TurnOn()` en `"SEND_ZIGBEE ON"`.
- `NestProtocol` podría traducirlo en `"POST /api/device/on"`.

##### ¿Cómo se Conecta?

- `DeviceCommunication` **usa** (tiene una referencia a) una instancia de `IDeviceProtocolImplementation`, utilizando la inyección de dependencias.
- Las clases concretas de dispositivos (`PhilipsLightAdapter`, `NestThermostatAdapter`) reciben una instancia de `DeviceCommunication`.

#### Resultado

Podemos tener muchos dispositivos diferentes (luces, termostatos, cámaras), todos usando diferentes formas de comunicación, sin que las clases de dispositivos sepan cómo se implementa la conexión.

```c#
var comm = new DeviceCommunication(new PhilipsProtocol());
var light = new PhilipsLightAdapter(comm);
light.TurnOn();
```

## Patrón Adapter

#### ¿Qué Resuelve?

Convierte la interfaz específica de cada fabricante en una interfaz **común**, que el sistema pueda usar sin importar el proveedor.

#### ¿Dónde está el Adapter?

Clases adaptadoras: Estas clases son **adaptadores concretos**.  
Se encargan de **recibir comandos estándar** (como `TurnOn()`) y **traducirlos** a lo que entiende el fabricante.
Cada adaptador tiene un objeto `DeviceCommunication`, que a su vez conoce el protocolo que se debe usar.

- `PhilipsLightAdapter`: traduce comandos como `TurnOn()` a llamadas genéricas `Send("ON")`.
- `NestThermostatAdapter`: traduce `SetTemperature(23.5)` a `Send("SET_TEMP 23.5")`.

##### Interfaces Adaptadas

- `ILightDevice`: interfaz esperada por la plataforma para luces.
- `IThermostatDevice`: interfaz esperada por la plataforma para termostatos.
  Ambas son implementadas por sus respectivos adaptadores (`PhilipsLightAdapter`, `NestThermostatAdapter`).

###### ¿Cómo se Conecta?

Cada adapter **recibe un objeto `DeviceCommunication`**, que oculta la implementación específica del protocolo. El adapter se comunica con el protocolo a través de esta capa intermedia.

#### Resultado

Puedes usar cualquier dispositivo del fabricante con una única interfaz esperada por el sistema.

```c#
var thermostat = new NestThermostatAdapter(new DeviceCommunication(new NestProtocol()));
thermostat.SetTemperature(21.0f);
```

## Patrón Observer

#### ¿Qué Resuelve?

Permite que la **plataforma central** observe los cambios de estado de los dispositivos sin estar acoplada directamente a ellos. Imagina que una luz se apaga o que un termostato cambia de temperatura.  
La plataforma necesita enterarse de estos cambios automáticamente, sin tener que estar preguntando a cada dispositivo.
El patrón **Observer** permite que una clase (la plataforma) **se suscriba** a eventos de los dispositivos y reaccione cuando algo cambia, sin estar directamente conectada a su lógica.

#### Participantes

##### Observer

- `IDeviceObserver`: interfaz con métodos `Update(deviceId, status)` y `RegisterDevice(device)`.
- `CentralPlatform`: implementa `IDeviceObserver`.

##### Sujeto Observado

Dispositivos (`PhilipsLightAdapter`, `NestThermostatAdapter`) tienen eventos:

- `StatusChanged` para luces.
- `TemperatureChanged` para termostatos.

#### Flujo

- `CentralPlatform.RegisterDevice(device)` agrega el dispositivo al sistema.
- Los dispositivos notifican mediante eventos cuando ocurre un cambio (`StatusChanged.Invoke("ON")`).
- El observador reacciona: `Update("light1", "ON")`.

#### Resultado

La plataforma puede reaccionar automáticamente a cualquier cambio de estado (encendido, temperatura, etc.) sin conocer los detalles del dispositivo.

### Flujo Completo de la Ejecución

1. Se define un protocolo de fabricante: `PhilipsProtocol` o `NestProtocol`.
2. Se inyecta en `DeviceCommunication`, que actúa como el puente.
3. `DeviceCommunication` se pasa al adaptador (`PhilipsLightAdapter`, `NestThermostatAdapter`).
4. El adaptador expone una interfaz uniforme: `TurnOn()`, `SetTemperature()`, etc.
5. Cuando el dispositivo cambia de estado, lanza un evento.
6. La plataforma central (`CentralPlatform`) recibe la notificación y actualiza el estado global.

### Ejemplo en CSharp

```c#
// Protocolo Philips
var philipsComm = new DeviceCommunication(new PhilipsProtocol());
var light = new PhilipsLightAdapter(philipsComm);

// Protocolo Nest
var nestComm = new DeviceCommunication(new NestProtocol());
var thermostat = new NestThermostatAdapter(nestComm);

// Registro en la plataforma
var platform = new CentralPlatform();
platform.RegisterDevice(light);
platform.RegisterDevice(thermostat);

// Uso de dispositivos
light.TurnOn();  // Dispara evento "StatusChanged"
thermostat.SetTemperature(22.5f); // Dispara evento "TemperatureChanged"
```

### Ejemplo en CSharp

```c#
// 1. Protocolo Nest y comunicación
var nestComm = new DeviceCommunication(new NestProtocol());

// 2. Adaptador para termostato
var thermostat = new NestThermostatAdapter(nestComm);

// 3. Plataforma central que observa
var platform = new CentralPlatform();
platform.RegisterDevice(thermostat);

// 4. Se cambia la temperatura
thermostat.SetTemperature(22.0f);

// 5. El dispositivo lanza evento y la plataforma reacciona
// Resultado: "Termostato Nest cambió a 22.0°C" (por ejemplo)
```
![image](https://github.com/user-attachments/assets/9b950950-0e23-4781-b563-4b2496a3ec47)

