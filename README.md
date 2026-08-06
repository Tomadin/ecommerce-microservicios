# ecommerce-microservicios

Sistema de microservicios para una tienda de electrodomésticos online.
Proyecto de portfolio · Java 21 · Spring Boot 3.5 · Spring Cloud.

> En construcción — Fase 0.

## Servicios

- `products-service` — catálogo y stock
- `cart-service` — carrito
- `orders-service` — órdenes, máquina de estados, pago mockeado
- `identity-service` — auth y emisión de JWT
- `notifications-service` — consume OrderPlaced
- `api-gateway` — único punto de entrada, valida JWT
- `eureka-server` — service discovery

## Cómo levantarlo

_(pendiente — docker-compose en Fase 4)_

## Decisiones de arquitectura

Ver carpeta `/docs/adr/`.
