# Gravitalia Architecture

This page describe Gravitalia event-driven architecture in production.
Production is fairly modular, which means that users self-hosting Gravitalia
can choose to remove certain microservices.

Removing certain microservices saves memory and processor time, but also
reduces the ability to monitor services effectively. Gravitalia has chosen to
limit metric events in any case, to guarantee a certain amount of "no-log."

In production, services MUST NOT be exposed. A reverse proxy (e.g., nginx) must
 also be added, particularly to distribute load, rate limits, and cache some
 routes. Gravitalia uses a tunnel (like Cloudflare Tunnel) to avoid exposing
 any ports outside world.

Services are written in Rust or Elixir, depending on requirements. RabbitMQ
consumers MUST support backpressure, with a single retry on failure. Events
MUST use CloudEvents AMPQ specification \[[1](
https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/amqp-protocol-binding.md)\].
## Account broker (Autha) architecture

```mermaid
graph TD
    subgraph Clients
        App[Web/mobile app]
        Third[Third party]
    end

    subgraph Service_Mesh ["Account service"]
        SA[Autha]
        Database[(PostgreSQL)]
        SI[Spinoza]
        SE[Maily]
        RMQ[(RabbitMQ)]
    end

    subgraph Messaging_Storage ["Storage"]
        S3[Stockage S3]
    end

    subgraph Telemetry ["Telemetry"]
        PROM((Prometheus))
        OTLP[OTLP collector]
        LOKI((Loki))
        JAEGER((Jaeger))
    end

    SA <-->|Save/get data| Database

    App -->|Authentification| SA
    SA -->|Upload Avatar| S3
    SI -->|Optimize image| S3

    SA -.->|Event: Welcome| RMQ
    S3 -.->|Event: ImageUploaded| RMQ
    RMQ -.->|Consume| SI
    RMQ -.->|Consume| SE

    SA -->|Traces| JAEGER
    SA & SI & SE & RMQ -->|Metrics| OTLP
    SA & SI & SE -->|Logs| OTLP
    
    OTLP -->|Export Metrics| PROM
    OTLP -->|Export Logs| LOKI
    OTLP -->|Export Traces| JAEGER

    style Telemetry fill:#f1f1f1,stroke:#333,stroke-dasharray: 5 5
    style Messaging_Storage fill:#e1f5fe,stroke:#01579b
    style JAEGER fill:#ffeb3b,color:#000
```

## Gravitalia Turms

```mermaid
graph TD
    subgraph Clients
        App[Desktop app]
    end

    subgraph Service_Account ["Account broker"]
        SA[Autha]
    end

    subgraph Service_Mesh ["Discovery services"]
        Discovery[Discovery server]
    end

    subgraph Telemetry ["Telemetry"]
        PROM((Prometheus))
        OTLP[OTLP collector]
        LOKI((Loki))
        JAEGER((Jaeger))
    end

    App -.->|Discover peers| Discovery
    Discovery -->|Login user| SA
  
    Discovery -->|Traces| JAEGER
    Discovery -->|Metrics| OTLP
    Discovery -->|Logs| OTLP
    
    OTLP -->|Export Metrics| PROM
    OTLP -->|Export Logs| LOKI
    OTLP -->|Export Traces| JAEGER

    style Telemetry fill:#f1f1f1,stroke:#333,stroke-dasharray: 5 5
    style JAEGER fill:#ffeb3b,color:#000
```
