# Airline Reservation System

A distributed airline booking platform built using a microservices architecture, where each service is independently developed, deployed, and scaled.

## Architecture Overview

```mermaid
graph TD
    A[API Gateway] --> B[Auth Service]
    A --> C[Flights & Search Service]
    A --> D[Reminder Service]
    B --> E[(MySQL - Auth DB)]
    C --> F[(MySQL - Flights DB)]
    D --> G[(Message Queue)]
```


