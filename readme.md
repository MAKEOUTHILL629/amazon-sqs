# AWS SQS como cola de mensajería [Diseño de infraestructura]

## EventBridge vs SNS como sistema de mensajería

### SNS vs EventBridge vs SQS

SNS, EventBridge y SQS se complementan en arquitecturas distribuidas. Mientras SQS actúa como sistema de colas, SNS y EventBridge funcionan como enrutadores de eventos. Se discuten sus capacidades de filtrado, orden, persistencia, escalabilidad y automatización, con ejemplos prácticos y estimaciones de costos.

- Clasificación de servicios

  - Enrutadores: SNS y EventBridge → distribuyen eventos a múltiples destinos.

  - Colas: SQS → almacenamiento temporal y entrega garantizada de mensajes.

- Concepto base

  - Productor (ej. tienda online) publica eventos en formato JSON.

  - El enrutador decide a qué colas SQS enviar cada evento.

  - Consumidores procesan mensajes de las colas de forma desacoplada.

- Arquitectura con SNS

  - Se crea un Topic (ej. domain-events).

  - Cada cola SQS se suscribe al topic.

  - Filtrado por Filter Policy en la suscripción (ej. solo eventos user_registered).

  - La lógica de filtrado vive en cada cola.

- Arquitectura con EventBridge

  - Se crea un Event Bus (ej. domain-events).

  - Se definen Rules con patrones de filtrado (ej. user.registered).

  - Cada regla tiene un Target (cola SQS u otro servicio).

  - La lógica de filtrado vive centralizada en el Event Bus.

- Arquitectura solo con SQS

  - Productor publica directamente en una cola.

  - Un “router manual” (Lambda, servidor, etc.) lee y reenvía a otras colas.

  - Desventaja: la lógica crece y se complica, se pierde escalabilidad y mantenibilidad.

- Beneficios de arquitecturas basadas en eventos

  - Desacoplamiento entre sistemas.

  - Escalabilidad independiente.

  - Resiliencia: si un consumidor cae, los mensajes quedan en cola.

  - Permite añadir nuevos consumidores sin tocar el sistema principal.

- Comparativa de funcionalidades

  - Orden garantizado: SNS (con FIFO) sí, EventBridge no.

  - Reintentos y DLQ: ambos vía SQS.

  - Mensajes programados: solo EventBridge.

  - Replay de eventos: EventBridge (con Archive activado).

  - Schema Registry: EventBridge (validación y versionado de eventos).

  - Coste: SNS suele ser más barato, pero EventBridge ofrece más features.

#### Arquitecturas

##### SNS + SQS

[Productor] → [SNS Topic] → [SQS Cola A]
                              [SQS Cola B]

- SNS recibe todos los eventos.

- Cada cola se suscribe y filtra lo que necesita.

- Filtrado distribuido (cada cola define su propia policy).

##### EventBridge + SQS

[Productor] → [Event Bus] → (Rule 1) → [SQS Cola A]
                              (Rule 2) → [SQS Cola B]

- EventBridge centraliza las reglas de filtrado.

- Cada regla define el patrón y el destino.

- Más ordenado para arquitecturas grandes.

##### Solo SQS con router manual

[Productor] → [SQS Cola Principal] → [Router Manual] → [SQS Cola A]
                                                  → [SQS Cola B]

- El router decide a dónde enviar cada mensaje.

- Mayor complejidad y riesgo de errores.

- Menos recomendable si se puede usar SNS o EventBridge.

### Checklist comparativa SNS vs EventBridge

#### Event Bus en AWS

| Característica                     | SNS                          | EventBridge                 |
|-----------------------------------|------------------------------|-----------------------------|
| Orden garantizado                | Sí (FIFO SNS+SQS)              | No                          |
| Evita duplicados                | Sí (FIFO SNS+SQS)              | No                          |
| Reintentos automáticos           | Sí (vía SQS)                  | Sí (vía SQS)                 |
| Dead Letter Queue (DLQ)          | Sí (vía SQS)                  | Sí (vía SQS)                 |
| Mensajes programados             | No                           | Sí                          |
| Persistencia de mensajes (replay) | No                           | Sí (con Archive activado)   |
| Rendimiento escalable         | Muy alto                     | Alto                        |
| Coste                           | Más barato                   | Más caro                    |
| Schema Registry                 | No                           | Sí (validación y versionado)|

- Orden garantizado: El orden solo se garantiza si se usa SNS FIFO (First In, First Out) junto con SQS FIFO. EventBridge no garantiza el orden de los eventos.
- Evitar duplicados: SNS FIFO (junto con SQS FIFO) evita duplicados, porque cada mensaje tiene un ID único. EventBridge no tiene esta característica.
- Reintentos automáticos: Ambos servicios permiten reintentos automáticos de entrega fallida, pero esto se gestiona a través de SQS.
- Dead Letter Queue (DLQ): Ambos servicios soportan DLQ mediante SQS para manejar mensajes que no se pueden entregar.
- Mensajes programados: Solo EventBridge permite programar mensajes para que se entreguen en un momento futuro.
- Persistencia de mensajes (replay): EventBridge permite archivar eventos y reproducirlos más tarde, mientras que SNS no tiene esta capacidad.
- Rendimiento escalable: SNS está diseñado para un rendimiento muy alto y puede manejar millones de de mensajes por segundo. EventBridge también es escalable, pero generalmente tiene un rendimiento algo menor.
- Coste: SNS suele ser más económico que EventBridge, especialmente para volúmenes altos de mensajes. EventBridge ofrece más funcionalidades, pero a un coste mayor.
- Schema Registry: EventBridge incluye un Schema Registry que permite validar y versionar los eventos, mientras que SNS no ofrece esta funcionalidad.

## Publica mensaje en EventBridge

### Setup: Primeros pasos con AWS EventBridge y SQS utilizando localstack

Se inicializa localstack con docker-compose:

```bash

docker-compose up -d
```

localstack inicia varios servicios, entre ellos EventBridge y SQS, es una solución ideal para desarrollo local.

Se crea un Event Bus y una cola SQS:

```bash
./configure.sh
```

Se publica un evento de prueba en el Event Bus:

```bash
aws events --endpoint-url http://localhost:4566 --region us-east-1 put-events --entries file://event.json
```

Se verifica que el mensaje ha llegado a la cola SQS:

```bash
aws sqs --endpoint-url http://localhost:4566 --region us-east-1 receive-message --queue-url http://localhost:4566/000000000000/send_welcome_email_on_user_registered
```bash   aws sqs --endpoint-url http://localhost:4566 --region us-east-1 receive-message --queue-url http://localhost:4566/000000000000/send_welcome_email_on_user_registered
```

### Publicación de mensajes en EventBridge

