# HiRider

HiRider is a ride-sharing platform built as a set of Go microservices with a Next.js web frontend, modeled after systems like Uber/Lyft. It supports trip preview and creation, real-time driver matching, live trip tracking over WebSockets, and Stripe-based payments.

![HiRider](assets/images/thumbnail.png)

## Architecture

The system is composed of independent Go services that communicate over gRPC (synchronous requests) and RabbitMQ (asynchronous events), fronted by an API Gateway that exposes HTTP and WebSocket endpoints to the web client.

```
                         ┌──────────────┐
                         │   Web (Next) │
                         └──────┬───────┘
                        HTTP / WebSocket
                                │
                         ┌──────▼───────┐
                         │  API Gateway │
                         └──┬────┬───┬──┘
                     gRPC   │    │   │  gRPC
                ┌───────────┘    │   └───────────┐
                │                │ RabbitMQ       │
         ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
         │ Trip Service│  │   RabbitMQ  │  │Driver Service│
         └──────┬──────┘  └──────┬──────┘  └─────────────┘
                │                │
                │         ┌──────▼──────┐
                └────────▶│Payment Svc  │────▶ Stripe
                          └─────────────┘
```

| Service | Responsibility | Protocols |
|---|---|---|
| [`api-gateway`](services/api-gateway) | Public entry point: HTTP + WebSocket for the web client, gRPC client to backend services | HTTP, WebSocket, gRPC (client) |
| [`trip-service`](services/trip-service) | Trip preview/creation, route calculation (OSRM), fare estimation, trip lifecycle | gRPC, RabbitMQ |
| [`driver-service`](services/driver-service) | Driver matching, trip request/accept/decline flow | gRPC, RabbitMQ |
| `payment-service` | Stripe checkout sessions, payment webhooks | gRPC, RabbitMQ, Stripe |
| [`shared`](shared) | Common libraries: messaging, tracing, Mongo, contracts, retry, env helpers | — |
| [`web`](web) | Next.js frontend (map preview, trip booking, live driver tracking) | — |

Services communicate asynchronously through RabbitMQ using a trip exchange and a payment exchange. See [`docs/architecture`](docs/architecture) for detailed sequence diagrams:

- [Trip creation flow](docs/architecture/trip-creation-flow.md)
- [RabbitMQ event flow](docs/architecture/rabbitmq-flow.md)

Distributed tracing is wired up via OpenTelemetry + Jaeger across HTTP, gRPC, and RabbitMQ.

## Tech stack

- **Backend:** Go, gRPC, Protocol Buffers, RabbitMQ, MongoDB
- **Frontend:** Next.js 15, React 19, TypeScript, Tailwind CSS, Leaflet (maps)
- **Payments:** Stripe
- **Routing:** OSRM (Open Source Routing Machine)
- **Observability:** OpenTelemetry, Jaeger
- **Infra/local dev:** Docker, Kubernetes, [Tilt](https://tilt.dev)

## Prerequisites

- [Go](https://go.dev) 1.23+
- [Node.js](web/.nvmrc) (see `.nvmrc`) and npm
- Docker
- A local Kubernetes cluster (e.g. [kind](https://kind.sigs.k8s.io) or Docker Desktop's Kubernetes) and `kubectl`
- [Tilt](https://tilt.dev)
- `protoc` with the Go and Go-gRPC plugins (only needed if you change the `.proto` files)

## Getting started

### 1. Configure secrets

Copy the Kubernetes secrets template and fill in real values:

```bash
cp infra/development/k8s/secrets.template.yaml infra/development/k8s/secrets.yaml
```

Edit `infra/development/k8s/secrets.yaml` and set:

- `MONGODB_URI` — your MongoDB connection string
- `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_KEY` — your Stripe API keys
- RabbitMQ credentials and the OSRM API URL can be left as the provided defaults for local development

### 2. Run the stack

With a Kubernetes context pointed at your local cluster:

```bash
tilt up
```

Tilt builds and deploys the API Gateway, Trip Service, Driver Service, Payment Service, web frontend, RabbitMQ, and Jaeger, and live-reloads on code changes.

Once running:

- Web app: [http://localhost:3000](http://localhost:3000)
- API Gateway: [http://localhost:8081](http://localhost:8081)
- RabbitMQ management UI: [http://localhost:15672](http://localhost:15672)
- Jaeger UI: [http://localhost:16686](http://localhost:16686)

### 3. Run the web frontend standalone (optional)

```bash
cd web
npm install
npm run dev
```

## Configuration reference

Services are configured via environment variables (see [`shared/env`](shared/env/env.go)), sourced from `app-config.yaml` and `secrets.yaml` in Kubernetes:

| Variable | Used by | Default |
|---|---|---|
| `ENVIRONMENT` | all services | `development` |
| `JAEGER_ENDPOINT` | all services | `http://jaeger:14268/api/traces` |
| `RABBITMQ_URI` | api-gateway, trip, driver, payment | `amqp://guest:guest@rabbitmq:5672/` |
| `HTTP_ADDR` | api-gateway | `:8081` |
| `MONGODB_URI` | trip-service | — |
| `OSRM_API` | trip-service | `http://router.project-osrm.org` |
| `STRIPE_SECRET_KEY` | payment-service | — |
| `STRIPE_SUCCESS_URL` | payment-service | `<APP_URL>?payment=success` |
| `STRIPE_CANCEL_URL` | payment-service | `<APP_URL>?payment=cancel` |
| `APP_URL` | payment-service | `http://localhost:3000` |

## Working with protobuf

Trip and driver service contracts are defined in [`proto/`](proto). After editing a `.proto` file, regenerate the Go bindings:

```bash
make generate-proto
```

## Repository layout

```
.
├── services/            # Go microservices (api-gateway, trip, driver, payment)
├── shared/               # Shared Go packages (messaging, db, tracing, contracts, types)
├── proto/                # Protobuf service definitions
├── web/                  # Next.js frontend
├── infra/                # Dockerfiles and Kubernetes manifests (development & production)
├── docs/architecture/    # Sequence and event-flow diagrams
├── tools/                # Codegen helper for scaffolding new services
├── Tiltfile              # Local dev orchestration
└── Makefile              # Protobuf codegen
```

## License

MIT — see [LICENSE](LICENSE).
