# game-core-service – Dron Wars Org

The **real-time game core microservice** for **Dron Wars**, a browser shoot 'em up heavily inspired by the NES classic *Abadox* (1989).

This service is the beating heart of the game: it handles server-authoritative logic, WebSocket synchronization, enemy AI, collision detection, power-up application, game state broadcasting, and room management for single-player and co-op modes.

## Why this service is key
- Manages high-frequency real-time updates (positions, bullets, hits) with low latency
- Ensures anti-cheat by validating moves server-side
- Enables scalable multiplayer (co-op rooms) using WebSockets + Redis caching
- Produces critical events (GameEnded, WaveCleared, AchievementUnlocked) via Kafka

## Features & Learning Objectives

- **Java 21+**: Virtual threads for handling thousands of concurrent WebSocket connections without thread blocking
- **Spring Boot 3.x**: WebSocket + STOMP protocol, Netty transport, Micrometer observability
- **Real-time sync**: STOMP destinations (`/topic/game/{roomId}`) for broadcasting game state
- **Server-authoritative**: Collision detection, enemy AI patterns (sine waves, homing), power-up logic
- **Room management**: In-memory + Redis-backed rooms for players (single or co-op)
- **Event-driven**: Kafka producers for game lifecycle events (integrates with progress-economy and leaderboard services)
- **Caching**: Redis for temporary game state (player positions, active bullets, room metadata)
- **Anti-cheat basics**: Validate client inputs (movement speed, fire rate) server-side
- **Testing**: JUnit 5 + Testcontainers (Redis + Kafka), WebSocket integration tests

## Tech Stack

- Java 21
- Spring Boot 3.3+ (latest stable)
- Spring WebSocket + STOMP
- Spring Kafka (producers)
- Spring Data Redis
- Lombok (optional)
- Maven
- Docker

## Main Endpoints & Protocols

| Type       | Path / Destination                  | Description                                      | Auth Required |
|------------|-------------------------------------|--------------------------------------------------|---------------|
| WS         | `/ws-game`                          | WebSocket endpoint (SockJS fallback)             | JWT           |
| STOMP SUB  | `/topic/game/{roomId}`              | Broadcast game state updates                     | -             |
| STOMP SEND | `/app/move`                         | Client sends player input (position, shoot)      | -             |
| STOMP SEND | `/app/join-room`                    | Join/create room (single or co-op)               | -             |
| REST       | `/api/game/health`                  | Actuator health check                            | No            |
| Kafka      | Topic: `game-events`                | Produces GameEnded, WaveCleared, etc.            | -             |

## Quick Start (Local Development)

### Prerequisites
- Java 21+
- Maven
- Docker & Docker Compose (Postgres, Redis, Kafka from monorepo)

### 1. Start shared infrastructure
```bash
# From Organization root or monorepo
docker-compose up -d redis kafka zookeeper
```

### 2. Run the service
```bash
cd game-core-service
mvn clean install
mvn spring-boot:run
```

Or with virtual threads:
```bash
java --enable-preview -XX:+UseZGC -jar target/game-core-service-0.0.1-SNAPSHOT.jar
```

→ WebSocket available at `ws://localhost:8082/ws-game` (configurable)

### 3. Example application.yml snippet
```yaml
spring:
  redis:
    host: localhost
    port: 6379
  kafka:
    bootstrap-servers: localhost:9092
  websocket:
    path: /ws-game
server:
  port: 8082
jwt:
  secret: ${JWT_SECRET:your-secret-key}
game:
  tick-rate-ms: 33          # ~30 updates/sec
  room:
    max-players: 2          # co-op support
```

## Project Structure

```
game-core-service/
├── src/
│   ├── main/
│   │   ├── java/com/dronwars/gamecore/
│   │   │   ├── config/          # WebSocketConfig, KafkaConfig, RedisConfig
│   │   │   ├── controller/      # GameWebSocketController (@MessageMapping)
│   │   │   ├── model/           # GameState, PlayerInput, Room (records!)
│   │   │   ├── service/         # GameLogicService, RoomService, AiService
│   │   │   ├── engine/          # CollisionEngine, PowerUpEngine
│   │   │   └── event/           # GameEvent (Kafka messages)
│   │   └── resources/
│   │       └── application.yml
├── pom.xml
└── Dockerfile
```

## Integration with Frontend (Phaser 3)

Frontend connects via STOMP.js:
```js
const client = new StompJs.Client({
  brokerURL: 'ws://localhost:8082/ws-game',
  connectHeaders: { Authorization: `Bearer ${jwt}` }
});

client.onConnect = () => {
  client.subscribe(`/topic/game/${roomId}`, message => {
    const state = JSON.parse(message.body);
    // Update Phaser scene: enemies, bullets, score...
  });
};

// Send player move
client.publish({
  destination: '/app/move',
  body: JSON.stringify({ x: player.x, y: player.y, shoot: true })
});
```

## Learning Notes

This service practices:
- High-concurrency WebSockets with virtual threads (no thread-per-connection)
- STOMP message handling & broadcasting
- Server-side game loop (tick-based updates)
- Redis pub/sub + caching for room state
- Kafka integration for decoupling (game events → economy & leaderboard)

Part of **dron-wars-org** → https://github.com/dron-wars-org

---

MIT License – personal learning/portfolio project  
Pablo – Don Torcuato, Buenos Aires – 2026
Está listo para que lo uses ya. Si querés agregar ejemplos de mensajes STOMP, un snippet de GameState DTO (record de Java 21), o más detalles sobre virtual threads en el WS config, decime y lo amplío.

Próximo paso: ¿README de **progress-economy-service**? ¿O arrancamos con código real (pom.xml, WebSocketConfig con virtual threads, o GameState record)? ¡Vos decís! 🛸🕹️
