# discovery-server

> Service registry for the SoftPOS platform, built on Netflix Eureka. Every other microservice registers itself here on startup and uses it to locate its dependencies dynamically — no hardcoded hostnames, no hardcoded ports.

![Spring Cloud Netflix](https://img.shields.io/badge/Spring%20Cloud-Netflix%20Eureka-brightgreen)
![Port](https://img.shields.io/badge/port-8761-blue)

---

## Table of contents

- [What this service does](#what-this-service-does)
- [What is Eureka, really](#what-is-eureka-really)
- [Eureka Server vs. Eureka Client](#eureka-server-vs-eureka-client)
- [Why service discovery instead of hardcoded URLs](#why-service-discovery-instead-of-hardcoded-urls)
- [How registration actually works](#how-registration-actually-works)
- [Self-preservation mode](#self-preservation-mode)
- [Running locally](#running-locally)
- [Running with Docker](#running-with-docker)
- [Configuration reference](#configuration-reference)
- [Verifying it works](#verifying-it-works)
- [Securing the dashboard](#securing-the-dashboard)
- [Common pitfalls](#common-pitfalls)

---

## What this service does

`discovery-server` provides the platform with a **clean, clear, always-up-to-date map of every running service instance** — what it's called, where it lives, and whether it's healthy. Instead of every service needing to know every other service's IP and port ahead of time, each service just asks one question — _"where is `transaction-service` right now?"_ — and gets a live answer.

Concretely, it gives the platform:

- **A single source of truth for service locations** — no service configuration file anywhere in the platform hardcodes another service's host/port.
- **Automatic handling of scaling and failure** — if you run three instances of `risk-service`, or one crashes and restarts on a new container IP, nothing else needs to be reconfigured. The registry reflects reality automatically.
- **A live topology view** — the dashboard at `:8761` is, at a glance, a map of exactly which services are up, how many instances of each, and since when.
- **A foundation for client-side load balancing** — because a caller can see _all_ healthy instances of a service, not just one, calls can be spread across them without a separate load balancer.

This is the first service to start in the platform (alongside `config-server`) because everything else depends on it to find its neighbors.

## What is Eureka, really

Eureka is **not an external product you install** — unlike Keycloak, Kafka, or Postgres, it doesn't ship as a Docker image you pull from a vendor. It's a **Java library** (`spring-cloud-starter-netflix-eureka-server`) that you add to a normal Spring Boot project. Adding one annotation, `@EnableEurekaServer`, turns that Spring Boot application into a service registry.

So `discovery-server` is a real microservice living in this repository, with its own `pom.xml`, its own Dockerfile, and its own deployment — it just happens to have almost no business logic of its own. Its entire job is bookkeeping: who's alive, where are they, and are they still healthy.

At its core, Eureka is a **REST-based registry with a heartbeat protocol**:

1. A service starts up and sends a `POST` to the registry: "I am `merchant-service`, I live at `10.0.1.4:8081`, mark me `UP`."
2. Every 30 seconds by default, that service sends a heartbeat: "still here."
3. If heartbeats stop for ~90 seconds, the registry marks that instance as down and eventually evicts it.
4. Any other service can `GET` the current list of healthy instances for a given service name at any time.

## Eureka Server vs. Eureka Client

This distinction is the one thing worth being completely clear on, because both roles are provided by the same Spring Cloud Netflix library family but do opposite jobs:

|                | Eureka **Server**                                                                                                    | Eureka **Client**                                                                                           |
| -------------- | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Dependency     | `spring-cloud-starter-netflix-eureka-server`                                                                         | `spring-cloud-starter-netflix-eureka-client`                                                                |
| Role           | **Is** the registry                                                                                                  | **Uses** the registry                                                                                       |
| Who is it      | Exactly one logical service in the platform: `discovery-server`                                                      | Every other service — `merchant-service`, `transaction-service`, `api-gateway`, all of them                 |
| What it does   | Accepts registrations, stores instance metadata, answers "who is X" queries, evicts dead instances                   | Registers itself on startup, sends heartbeats, fetches the registry to resolve other services' locations    |
| Annotation     | `@EnableEurekaServer`                                                                                                | None needed in modern Spring Cloud — just having the dependency on the classpath triggers auto-registration |
| Special config | `register-with-eureka: false` and `fetch-registry: false` — the server doesn't need to register with or query itself | `service-url.defaultZone` pointing at the server's address                                                  |

In short: **there is one Eureka Server in the whole platform (this service), and every single other microservice is a Eureka Client of it** — including, technically, `api-gateway`, which is simultaneously a client of Eureka (to resolve routes) and the entry point for external traffic.

## Why service discovery instead of hardcoded URLs

In a monolith, or a small setup with two or three services, hardcoding `http://merchant-service-host:8081` in a config file is tempting and "works." It stops working the moment you:

- Scale a service to multiple instances (which one do you hardcode?)
- Move a service to a new host or container (every dependent config needs updating)
- Run the same platform across dev/staging/prod with different topologies (you'd need separate hardcoded configs per environment)

Service discovery decouples "what a service is called" from "where it currently lives." Callers use Eureka's client-side load-balancing (via `spring-cloud-starter-loadbalancer`) with a logical name like `transaction-service`, and the actual IP resolution happens transparently underneath.

## How registration actually works

1. A client service starts and reads its `eureka.client.service-url.defaultZone` — the address of this `discovery-server`.
2. On startup, it sends a registration request containing its `spring.application.name`, host, port, and a health check URL.
3. `discovery-server` stores this in its in-memory registry and starts expecting heartbeats.
4. The client polls back periodically (default every 30s) to refresh its local copy of the full registry, so it also knows about every _other_ registered service without querying the server on every single call.
5. When the client shuts down cleanly, it sends a deregistration request. If it crashes, the server relies on missed heartbeats to eventually evict it.

## Self-preservation mode

Worth understanding before it confuses you in local development: Eureka has a built-in **self-preservation mode**. If the server suddenly stops receiving heartbeats from a large percentage of registered clients at once, it assumes this is more likely a _network partition_ than every single service having actually crashed simultaneously — and it stops evicting instances, keeping the (possibly stale) registry as-is rather than wiping it out.

You'll typically see a banner in the dashboard: _"EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP..."_ — this is expected and harmless in local development (where you're constantly stopping/starting services manually and triggering exactly the pattern that looks like a partition). It matters more in production, where it's a genuinely useful safety net against a registry wiping out your entire topology due to a transient network blip.

## Running locally

```bash
cd infra/discovery-server
./mvnw spring-boot:run
```

Then open **http://localhost:8761** — you should see the Eureka dashboard with an empty _"Instances currently registered with Eureka"_ table. That's correct at this stage; nothing has connected yet.

## Running with Docker

```bash
docker compose up discovery-server
```

The service builds from its own `Dockerfile` (multi-stage: Maven build → slim JRE runtime image) rather than pulling a pre-built image, since — as explained above — there is no official "Eureka server" product image; this **is** your application.

## Configuration reference

| Property                                 | Purpose                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `server.port`                            | Port the registry listens on (`8761` by convention)                                                              |
| `spring.application.name`                | Name shown for this service in logs/tracing — `discovery-server`                                                 |
| `eureka.client.register-with-eureka`     | Set `false` — the server should not register with itself                                                         |
| `eureka.client.fetch-registry`           | Set `false` — the server doesn't need to fetch a registry from itself                                            |
| `eureka.instance.hostname`               | Hostname clients use to reach this server (`localhost` locally, `discovery-server` inside Docker)                |
| `eureka.server.enable-self-preservation` | Can be disabled (`false`) in local/dev to see immediate eviction of stopped services — keep `true` in production |

## Verifying it works

1. Start `discovery-server` alone and confirm the dashboard loads at `:8761` with an empty instance table.
2. Start any client service (e.g. a placeholder service with just the Eureka Client dependency and a matching `service-url`).
3. Refresh the dashboard — the new service should appear under **"Instances currently registered with Eureka"** with status `UP`, listed under its `spring.application.name` in capital letters (e.g. `MERCHANT-SERVICE`).
4. Stop that client service and wait roughly 90 seconds — it should disappear from the registry once its heartbeat lease expires.

## Securing the dashboard

By default, the Eureka dashboard and registry API are unauthenticated — anyone who can reach port `8761` can see your entire service topology (names, hosts, ports), which is more information disclosure than a fintech project should tolerate even in development. Add `spring-boot-starter-security` and basic auth credentials sourced from environment variables (never hardcoded), and update every client's `defaultZone` URL to include those credentials.

## Common pitfalls

- **Forgetting `register-with-eureka: false` / `fetch-registry: false`** on the server itself — it still works, but the server needlessly treats itself as a client, adding noise to logs and the dashboard.
- **Using `localhost` inside Docker Compose** — containers can't reach each other via `localhost`; client services need `eureka.instance.prefer-ip-address: true` and a `defaultZone` pointing at the Docker service name (`discovery-server`), not `localhost`.
- **Expecting instant deregistration** — by default there's up to a ~90 second delay between a service dying and Eureka evicting it (30s heartbeat interval × 3 missed beats). This is tunable but exists for a reason: it avoids evicting a service over a single missed heartbeat during a brief GC pause or network blip.
- **Mistaking self-preservation warnings for a bug** — see the section above; it's a deliberate safety mechanism, not a malfunction.
