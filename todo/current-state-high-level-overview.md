# Eivo Platform - Current State

## Document Purpose

This document provides a high-level overview of the Eivo platform's
current state, organized by functional areas: Product, Architecture,
Development, QA, Operations, and Security.

# Product

## Current State

The platform provides capabilities for delivering educational
experiences: Learning (instructional content with exercises), Practice
(individual, group, and competitive sessions), and Evaluation (writing
and programming assessments with AI feedback). A reference
implementation (Lingv) demonstrates these capabilities working together
but is not intended as a commercial product.

## What's Missing

**A defined starter experience is needed.** The platform has working
capabilities but no product definition for what the first commercial
experience should be. This requires deciding what type of educational
experience to build. The starter experience must define its target
audience, learning objectives, content scope, user journey, feature
prioritization from available capabilities, pricing model, and
go-to-market strategy.

**Content strategy needs definition.** Initial content for the starter
experience must be created. This includes determining content depth,
breadth, quality standards, acquisition approach, and ongoing content
development plans.

**Mobile strategy is undefined.** Mobile support needs and priorities
must be determined based on the chosen audience and use cases.

**User experience standards need definition.** Product-level
expectations for error handling, user feedback, accessibility, and
overall user experience quality must be established for a commercial
offering.

# Architecture

The platform architecture is defined and documented across three layers.
The Infrastructure Layer consists of Kubernetes-based services including
Cloud API (NestJS modular monolith), Realtime services (Socket.IO), and
Environment Operator (Kubernetes operator in Go). The SDK Layer contains
Core SDK for business logic and Facets SDK for navigation and UI
framework. The Experience Layer is where applications are built by
composing SDK capabilities.

The Cloud API follows a modular monolith pattern with modules for
Editorial (content generation), Assistance (learner feedback), Social
(organization and activity), Gaming (leaderboards and competition), and
Identity (profiles and privacy). Each module maintains its own data
storage and exposes REST APIs with typed contracts.

Core SDK implements capability business logic: Learning for content
delivery, Trials for practice sessions (SoloTrial, GroupTrial),
Challenges for evaluation workflows (WritingChallenge,
EnvironmentChallenge), and generation engines (Crafter, Foundry) for
AI-powered content creation. State is managed in Redis for sessions and
PostgreSQL for persistent data.

Facets SDK provides declarative navigation through YAML definitions and
feature providers that delegate rendering to React components. The
framework uses a provider registry to match routes and render
appropriate UI.

Environment provisioning uses a Kubernetes operator that watches
Environment CRDs and provisions complete infrastructure including
compute, storage, IDE (Theia), and networking with automatic TTL-based
cleanup.

The architecture supports the platform's functional requirements and
technical constraints. It is cloud-native, enables AI integration across
multiple providers, supports isolated execution environments, and
provides real-time coordination for collaborative features.

# Development

## SDK Layer

The Core SDK implements business logic for all platform capabilities
(Learning, Trials, Challenges, content generation). The Facets SDK
provides navigation framework and UI components but is embedded in the
Lingv application rather than existing as a standalone package,
preventing its use in building other experiences.

## Services Layer

The Cloud API provides REST APIs for content generation (Editorial
module). Social and Gaming modules exist with basic functionality only.
The Realtime service handles WebSocket coordination for group features.
The Environment Operator provisions Kubernetes-based coding
environments.

## Infrastructure & Tooling

The codebase is organized as a PNPM monorepo. Build processes compile
TypeScript locally then create Docker images. Deployment uses Makefiles.
All deployment is manual - there is no CI/CD pipeline.

## What's Missing

**Facets SDK extraction** is the critical need. It must be separated
from Lingv into a standalone package to enable building new experiences.

**CSS architecture** requires restructuring. Styling is mixed with
component logic. Migration to pure CSS with theming support is needed.

**SDK implementations** for Social, Gaming, and Assistance capabilities
need to be built in Core SDK and Facets SDK.

**Assistance module** needs to be implemented for agentic experiences in
Cloud API.

**Social and Gaming modules** in Cloud API have only basic functionality
and need completion.

**Testing infrastructure** is minimal. Comprehensive automated test
suites need implementation.

**API documentation** is incomplete.

**Developer documentation** and setup guides are limited.

# QA

## Current State

Testing is primarily manual. Some unit tests exist in Core SDK
components but coverage is minimal. Basic integration tests cover a few
scenarios. There are no end-to-end automated tests. Performance testing
has not been conducted - behavior under load is unknown. Security
testing is not part of the development process.

Error handling is inconsistent across the platform. Technical error
messages are exposed to users rather than friendly explanations. Error
boundaries are incomplete. Loading states vary across features - some
provide feedback, others don't. Accessibility features are partial -
ARIA attributes, keyboard navigation, and screen reader support have
gaps.

## What's Missing

A comprehensive automated test suite is required covering unit tests for
all Core SDK capabilities, integration tests for API contracts and
service interactions, and end-to-end tests for critical user workflows.
Performance testing must be implemented to establish benchmarks and
validate behavior under expected load.

Error handling needs to be standardized throughout the platform with
user-friendly messages, proper error boundaries, and graceful
degradation. All features need consistent loading states and progress
indicators. Accessibility must be completed with full ARIA support,
comprehensive keyboard navigation, and screen reader compatibility.

Quality processes are absent - no automated testing in deployment
pipelines, no regression testing suite, no cross-browser testing, no
mobile device testing. These processes need establishment along with the
testing infrastructure.

# Operations

## Current State

The platform runs on Kubernetes with manual deployment processes.
CloudNativePG provides PostgreSQL database management with solid
reliability. Redis handles session state and trial queue storage.
Zitadel provides authentication via OIDC/JWT. The Environment Operator
provisions coding environments automatically with TTL-based cleanup.

Deployment uses Makefiles to build Docker images, push to registry, and
apply Kubernetes manifests. Database schemas are managed through
migration scripts executed manually. Seeders populate configuration,
prompts, and environment assets to persistent volumes. All deployment
operations are manual without automation.

## What's Missing

A complete observability stack is required including centralized logging
(ELK or Loki), metrics collection and visualization
(Prometheus/Grafana), distributed tracing (Jaeger or Tempo), and
alerting rules for operational issues. Currently, logging is scattered
and there is no comprehensive monitoring.

Backup and recovery procedures are manual. Automated backup systems are
needed for databases and critical data stores along with tested recovery
procedures. Disaster recovery planning and documentation is absent.

CI/CD pipeline infrastructure must be implemented to automate building,
testing, and deployment. This includes automated test execution before
deployment, staged rollout capabilities, and rollback procedures.

Production readiness for PostgreSQL and Redis needs to be assessed and
configured. Kubernetes resource limits and quotas need definition based
on expected load. Scaling policies are undefined.

Infrastructure monitoring and health checks are incomplete. Service
health endpoints need standardization. Resource utilization monitoring
needs establishment. Performance profiling tools are not in place.

# Security

## Current State

Authentication uses Zitadel with OIDC/JWT tokens. Basic authentication
flow works for user access. Authorization is minimal - basic role checks
exist but comprehensive permission systems are not implemented.

The platform runs on Kubernetes which provides container isolation.
Environment Operator creates isolated coding environments for
challenges. Network policies and pod security contexts are defined in
some places but not consistently applied.

## What's Missing

Rate limiting is not implemented - APIs and services have no protection
against abuse or DDoS attacks. Audit logging is minimal - comprehensive
security event logging for compliance is absent. Security scanning is
not part of the development or deployment process - no vulnerability
scanning for dependencies or container images.

LLM security measures need to be implemented - firewalls and protections
against prompt injection, jailbreaking, and other LLM-specific attacks.

A comprehensive authorization system is needed with fine-grained
permissions based on access tokens and role-based access control
throughout the platform. Security hardening must be applied including
consistent network policies, pod security policies, secrets management
practices, and certificate automation.

Security testing must be integrated into development workflows including
dependency vulnerability scanning, container image scanning, penetration
testing, and security audits. Compliance requirements (if any) need
assessment and implementation.

Privacy controls beyond basic authentication are not implemented. Data
protection measures, consent management, and regulatory compliance
capabilities (GDPR, CCPA if applicable) need development if enterprise
customers are targeted.

# Summary by Area

**Product**: Core capabilities functional in one experience (Lingv).
Need framework separation to enable multiple experiences, complete
incomplete capabilities (flashcards, social, gaming), add more exercise
types, improve mobile support.

**Development**: Solid technical foundation with working SDK and
services. Need to separate Facets from Lingv, migrate to pure CSS,
implement CI/CD, complete documentation.

**QA**: Primarily manual testing with minimal automation. Need
comprehensive test suite (unit, integration, e2e), performance testing,
standardized error handling, complete accessibility features, establish
quality processes.

**Operations**: Manual deployment on functional Kubernetes
infrastructure. Need observability stack (logging, monitoring, tracing,
alerting), automated backup/recovery, CI/CD pipeline, resource
optimization, health check standardization, scaling policies.

**Security**: Basic authentication working, minimal authorization. Need
rate limiting, audit logging, security scanning, comprehensive
authorization, security hardening, security testing integration, privacy
controls for enterprise readiness.
