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

Para publicar mensajes en EventBridge, se utiliza el comando `put-events` de la CLI de AWS. Este comando permite enviar uno o más eventos a un bus de eventos específico.

```bash
aws events --endpoint-url http://localhost:4566 --region us-east-1 put-events --entries file://event.json
```

```bash
aws sqs --endpoint-url http://localhost:4566 --region us-east-1 receive-message --queue-url http://localhost:4566/000000000000/send_welcome_email_on_user_registered

```

Para hacerlo desde un lenguaje de programación, se puede usar el SDK de AWS. A continuación, un ejemplo en Java Spring Wwebflux:

```java
import org.springframework.web.reactive.function.client.WebClient;  

public class EventBridgePublisher {

    private final WebClient webClient;

    public EventBridgePublisher(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.baseUrl("http://localhost:4566").build();
    }

    public void publishEvent(String eventJson) {
        webClient.post()
                .uri("/2015-10-07/events")
                .bodyValue(eventJson)
                .retrieve()
                .bodyToMono(String.class)
                .subscribe(response -> System.out.println("Event published: " + response));
    }
}

```

Los eventos tienen el siguiente formato JSON:

```json
[
  {
    "Source": "my.application",
    "DetailType": "user.registered",
    "Detail": "{\"userId\": \"123\", \"email\": \"user@example.com\"}"
  }
]
```

Cada evento tiene unos valores que son obligatorios y otros que varian dependiendo del evento:

- `Source`: Identifica la fuente del evento (ej. "my.application").
- `DetailType`: Tipo de evento (ej. "user.registered").
- `Detail`: Detalles específicos del evento en formato JSON (ej. información del usuario).
- `EventBusName`: Nombre del bus de eventos (opcional, si no se especifica se usa el bus por defecto).
- `Time`: Marca temporal del evento (opcional, si no se especifica se usa la hora actual).
- `Resources`: Lista de recursos relacionados con el evento (opcional).
- `TraceHeader`: Información de rastreo para correlacionar eventos (opcional).
- `Id`: Identificador único del evento (opcional, si no se especifica se genera uno automáticamente).
- `Version`: Versión del esquema del evento (opcional).
- `Region`: Región de AWS donde se envía el evento (opcional, si no se especifica se usa la región configurada).
- `Account`: Cuenta de AWS desde la cual se envía el evento (opcional, si no se especifica se usa la cuenta configurada).

### Fallback a la publicación de tus eventos en AWS EventBridge

Si por alguna razón el envio del mensaje falla, se propone guardar el evento en un tabla de base de datos para reintentar el envío más tarde, se propone crear un endpoint REST para reintentar el envío de eventos fallidos.

```java
package com.example.controller;

import com.example.service.EventPublisherService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private EventPublisherService eventPublisherService;
    
    @PostMapping("/register")
    public Mono<ResponseEntity<Map<String, String>>> registerUser(@RequestBody Map<String, String> userData) {
        // Lógica de registro de usuario...
        
        // Publicar evento de usuario registrado
        Map<String, Object> eventDetail = Map.of(
            "userId", userData.get("userId"),
            "email", userData.get("email"),
            "registrationDate", java.time.LocalDateTime.now().toString()
        );
        
        return eventPublisherService.publishEvent("my.application", "user.registered", eventDetail)
                .map(success -> {
                    if (success) {
                        return ResponseEntity.ok(Map.of("message", "Usuario registrado y evento enviado"));
                    } else {
                        return ResponseEntity.ok(Map.of("message", "Usuario registrado, evento guardado para reintento"));
                    }
                });
    }
}


package com.example.controller;

import com.example.entity.FailedEvent;
import com.example.repository.FailedEventRepository;
import com.example.service.EventPublisherService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/failed-events")
public class FailedEventController {
    
    @Autowired
    private FailedEventRepository failedEventRepository;
    
    @Autowired
    private EventPublisherService eventPublisherService;
    
    @GetMapping
    public ResponseEntity<List<FailedEvent>> getAllFailedEvents() {
        List<FailedEvent> events = failedEventRepository.findAll();
        return ResponseEntity.ok(events);
    }
    
    @GetMapping("/pending")
    public ResponseEntity<List<FailedEvent>> getPendingEvents() {
        List<FailedEvent> events = failedEventRepository.findPendingEvents();
        return ResponseEntity.ok(events);
    }
    
    @GetMapping("/retryable")
    public ResponseEntity<List<FailedEvent>> getRetryableEvents() {
        List<FailedEvent> events = failedEventRepository.findRetryableEvents();
        return ResponseEntity.ok(events);
    }
    
    @PostMapping("/{id}/retry")
    public Mono<ResponseEntity<Map<String, Object>>> retryEvent(@PathVariable Long id) {
        return Mono.fromCallable(() -> failedEventRepository.findById(id))
                .flatMap(optionalEvent -> {
                    if (optionalEvent.isEmpty()) {
                        return Mono.just(ResponseEntity.notFound().<Map<String, Object>>build());
                    }
                    
                    FailedEvent event = optionalEvent.get();
                    return eventPublisherService.retryFailedEvent(event)
                            .map(success -> {
                                Map<String, Object> response = Map.of(
                                    "success", success,
                                    "message", success ? "Evento reenviado exitosamente" : "Fallo al reenviar evento",
                                    "eventId", id,
                                    "retryCount", event.getRetryCount()
                                );
                                return ResponseEntity.ok(response);
                            });
                });
    }
    
    @PostMapping("/retry-all")
    public Mono<ResponseEntity<Map<String, Object>>> retryAllFailedEvents() {
        List<FailedEvent> retryableEvents = failedEventRepository.findRetryableEvents();
        
        return Flux.fromIterable(retryableEvents)
                .flatMap(eventPublisherService::retryFailedEvent)
                .collectList()
                .map(results -> {
                    long successCount = results.stream().mapToLong(success -> success ? 1 : 0).sum();
                    long failedCount = results.size() - successCount;
                    
                    Map<String, Object> response = Map.of(
                        "totalProcessed", results.size(),
                        "successCount", successCount,
                        "failedCount", failedCount,
                        "message", String.format("Procesados %d eventos: %d exitosos, %d fallidos", 
                                 results.size(), successCount, failedCount)
                    );
                    return ResponseEntity.ok(response);
                });
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Map<String, String>> deleteFailedEvent(@PathVariable Long id) {
        if (failedEventRepository.existsById(id)) {
            failedEventRepository.deleteById(id);
            return ResponseEntity.ok(Map.of("message", "Evento eliminado exitosamente"));
        }
        return ResponseEntity.notFound().build();
    }
}

package com.example.service;

import com.example.entity.FailedEvent;
import com.example.repository.FailedEventRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import software.amazon.awssdk.services.sqs.SqsAsyncClient;
import software.amazon.awssdk.services.sqs.model.SendMessageRequest;
import software.amazon.awssdk.services.sqs.model.SendMessageResponse;

import java.time.LocalDateTime;
import java.util.Map;

@Service
public class EventPublisherService {
    
    private static final Logger logger = LoggerFactory.getLogger(EventPublisherService.class);
    
    @Autowired
    private SqsAsyncClient sqsAsyncClient;
    
    @Autowired
    private FailedEventRepository failedEventRepository;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Value("${aws.sqs.queue.url}")
    private String defaultQueueUrl;
    
    public Mono<Boolean> publishEvent(String eventSource, String detailType, 
                                    Map<String, Object> eventDetail) {
        return publishEvent(eventSource, detailType, eventDetail, defaultQueueUrl);
    }
    
    public Mono<Boolean> publishEvent(String eventSource, String detailType, 
                                    Map<String, Object> eventDetail, String queueUrl) {
        try {
            String detailJson = objectMapper.writeValueAsString(eventDetail);
            
            SendMessageRequest messageRequest = SendMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .messageBody(createEventMessage(eventSource, detailType, detailJson))
                    .build();
            
            return Mono.fromFuture(sqsAsyncClient.sendMessage(messageRequest))
                    .map(SendMessageResponse::messageId)
                    .doOnNext(messageId -> logger.info("Evento enviado exitosamente. MessageId: {}", messageId))
                    .map(messageId -> true)
                    .onErrorResume(error -> {
                        logger.error("Error al enviar evento a SQS: {}", error.getMessage(), error);
                        return saveFailedEvent(eventSource, detailType, detailJson, queueUrl, error.getMessage())
                                .map(savedEvent -> false);
                    });
                    
        } catch (Exception e) {
            logger.error("Error al serializar evento: {}", e.getMessage(), e);
            return saveFailedEvent(eventSource, detailType, eventDetail.toString(), queueUrl, e.getMessage())
                    .map(savedEvent -> false);
        }
    }
    
    private String createEventMessage(String source, String detailType, String detail) {
        try {
            Map<String, Object> event = Map.of(
                "Source", source,
                "DetailType", detailType,
                "Detail", detail,
                "Time", LocalDateTime.now().toString()
            );
            return objectMapper.writeValueAsString(event);
        } catch (Exception e) {
            throw new RuntimeException("Error creando mensaje del evento", e);
        }
    }
    
    private Mono<FailedEvent> saveFailedEvent(String eventSource, String detailType, 
                                            String detail, String queueUrl, String errorMessage) {
        return Mono.fromCallable(() -> {
            FailedEvent failedEvent = new FailedEvent(eventSource, detailType, detail, queueUrl, errorMessage);
            FailedEvent savedEvent = failedEventRepository.save(failedEvent);
            logger.info("Evento fallido guardado en BD con ID: {}", savedEvent.getId());
            return savedEvent;
        });
    }
    
    public Mono<Boolean> retryFailedEvent(FailedEvent failedEvent) {
        if (failedEvent.getRetryCount() >= failedEvent.getMaxRetries()) {
            failedEvent.setStatus(FailedEvent.EventStatus.FAILED_MAX_RETRIES);
            failedEventRepository.save(failedEvent);
            return Mono.just(false);
        }
        
        failedEvent.setStatus(FailedEvent.EventStatus.RETRYING);
        failedEvent.setRetryCount(failedEvent.getRetryCount() + 1);
        failedEvent.setLastRetryAt(LocalDateTime.now());
        failedEventRepository.save(failedEvent);
        
        SendMessageRequest messageRequest = SendMessageRequest.builder()
                .queueUrl(failedEvent.getQueueUrl())
                .messageBody(createEventMessage(failedEvent.getEventSource(), 
                           failedEvent.getDetailType(), failedEvent.getDetail()))
                .build();
        
        return Mono.fromFuture(sqsAsyncClient.sendMessage(messageRequest))
                .map(response -> {
                    failedEvent.setStatus(FailedEvent.EventStatus.SUCCESS);
                    failedEventRepository.save(failedEvent);
                    logger.info("Reintento exitoso para evento ID: {}", failedEvent.getId());
                    return true;
                })
                .onErrorResume(error -> {
                    failedEvent.setStatus(FailedEvent.EventStatus.PENDING);
                    failedEvent.setErrorMessage(error.getMessage());
                    failedEventRepository.save(failedEvent);
                    logger.error("Reintento fallido para evento ID: {}", failedEvent.getId(), error);
                    return Mono.just(false);
                });
    }
}

package com.example.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "failed_events")
public class FailedEvent {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "event_source", nullable = false)
    private String eventSource;
    
    @Column(name = "detail_type", nullable = false)
    private String detailType;
    
    @Column(name = "detail", columnDefinition = "TEXT")
    private String detail;
    
    @Column(name = "event_bus_name")
    private String eventBusName;
    
    @Column(name = "queue_url")
    private String queueUrl;
    
    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;
    
    @Column(name = "retry_count", nullable = false)
    private Integer retryCount = 0;
    
    @Column(name = "max_retries", nullable = false)
    private Integer maxRetries = 3;
    
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "last_retry_at")
    private LocalDateTime lastRetryAt;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private EventStatus status = EventStatus.PENDING;
    
    // Constructors
    public FailedEvent() {
        this.createdAt = LocalDateTime.now();
    }
    
    public FailedEvent(String eventSource, String detailType, String detail, 
                      String queueUrl, String errorMessage) {
        this();
        this.eventSource = eventSource;
        this.detailType = detailType;
        this.detail = detail;
        this.queueUrl = queueUrl;
        this.errorMessage = errorMessage;
    }
    
    // Getters y Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getEventSource() { return eventSource; }
    public void setEventSource(String eventSource) { this.eventSource = eventSource; }
    
    public String getDetailType() { return detailType; }
    public void setDetailType(String detailType) { this.detailType = detailType; }
    
    public String getDetail() { return detail; }
    public void setDetail(String detail) { this.detail = detail; }
    
    public String getEventBusName() { return eventBusName; }
    public void setEventBusName(String eventBusName) { this.eventBusName = eventBusName; }
    
    public String getQueueUrl() { return queueUrl; }
    public void setQueueUrl(String queueUrl) { this.queueUrl = queueUrl; }
    
    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
    
    public Integer getRetryCount() { return retryCount; }
    public void setRetryCount(Integer retryCount) { this.retryCount = retryCount; }
    
    public Integer getMaxRetries() { return maxRetries; }
    public void setMaxRetries(Integer maxRetries) { this.maxRetries = maxRetries; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getLastRetryAt() { return lastRetryAt; }
    public void setLastRetryAt(LocalDateTime lastRetryAt) { this.lastRetryAt = lastRetryAt; }
    
    public EventStatus getStatus() { return status; }
    public void setStatus(EventStatus status) { this.status = status; }
    
    public enum EventStatus {
        PENDING, RETRYING, SUCCESS, FAILED_MAX_RETRIES
    }
}

```

## Arquitectura de colas de mensajeria

### Todos los mensajes a una sola cola



