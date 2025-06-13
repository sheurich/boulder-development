# boulder-development

This repository provides a development reference and implementation guide for AI agents and developers working on the Boulder ACME Certificate Authority project. Boulder implements a security-oriented microservices architecture for certificate issuance and management, with a strong emphasis on security, scalability, and maintainability.

## Overview

The purpose of this repository is to enable AI-assisted feature development, debugging, and maintenance across Boulder’s services. It outlines Boulder’s architecture, service responsibilities, development patterns, and provides practical implementation workflows for adding new features or extending existing functionality.

## Key Features

- **AI Agent Development Guide:** Structured information to help AI agents and developers understand Boulder’s microservices, architecture, and coding patterns.
- **Service-Oriented Architecture:** Descriptions of each Boulder service, including Web Front End (WFE2), Registration Authority (RA), Validation Authority (VA), Certificate Authority (CA), Storage Authority (SA), Publisher, Nonce Service, and more.
- **Development Patterns:** Examples and templates for rate-limiting, service constructors, error handling, database transactions, and gRPC service implementations.
- **Implementation Workflows:** Step-by-step instructions for common development tasks such as adding a new ACME challenge, updating database schemas, and integrating feature flags.
- **Testing and Environment Setup:** Guidance on using Docker Compose for local development, running unit and integration tests, and troubleshooting.

## Architecture

Boulder uses a microservices approach with clear separation of security contexts. Inter-service communication is handled via gRPC and protocol buffers, and persistent data is managed centrally by the Storage Authority. Feature rollouts are controlled with feature flags, and all changes must adhere to strict security and code quality guidelines.

## Getting Started

1. **Clone the repository:**
   ```bash
   git clone https://github.com/sheurich/boulder-development.git
   cd boulder-development
   ```
2. **Set up the development environment:**
   - Use Docker Compose for local builds and service orchestration.
   - See the [AGENTS.md](./AGENTS.md) for detailed setup and workflow instructions.

3. **Development and Testing:**
   - Follow the documented patterns for implementing new features.
   - Use `./t.sh` and `./tn.sh` to run tests as described in the guide.

## Documentation

- See [AGENTS.md](./AGENTS.md) for a comprehensive development guide, including architectural diagrams, service roles, key constraints, coding patterns, and step-by-step instructions for new feature implementation and testing.

## Contributions

Please follow Boulder’s established development and security practices when contributing. All new features should extend existing services, use gRPC interfaces, and be gated behind feature flags. See the development guide for more details.

---

For more information, refer to [AGENTS.md](./AGENTS.md).
