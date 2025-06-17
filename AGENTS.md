# AI Agent Development Guide for Boulder

## Goals and Context

This document serves as a comprehensive reference for AI agents assisting with feature development, debugging, and maintenance tasks within the Boulder ACME Certificate Authority project. Boulder is a production-grade implementation of the ACME protocol (RFC 8555) used by Let's Encrypt to issue millions of certificates.

## Quick Start Checklist

**Before starting any Boulder development task:**

1. **Read this entire document** - Don't skip sections; the routing matrix and patterns are essential
2. **Validate environment setup** - Run `docker compose build --pull` if you encounter Docker issues
3. **Check documentation first** - When encountering issues, always check `*.md` docs in boulder-development repo before proceeding
4. **Use the decision matrix** - Follow the service routing guide below to identify which services to modify
5. **Test with proper syntax** - Use `./t.sh --unit --filter=TestName` (note the equals sign in filter syntax)

### Purpose

- **Enable AI-assisted development**: Provide structured information that AI agents can reliably interpret to understand Boulder's architecture, services, and development patterns
- **Accelerate feature implementation**: Guide AI agents through Boulder's microservices architecture to implement new features correctly
- **Improve debugging efficiency**: Help AI agents quickly identify the right services and code paths when troubleshooting issues
- **Maintain code quality**: Ensure AI-generated code follows Boulder's established patterns and security practices

### Key Constraints for AI Development

- **No standalone services**: New features should extend existing services rather than creating new microservices
- **gRPC interfaces**: All service-to-service communication must use defined protocol buffer interfaces
- **Database migrations**: Schema changes require proper migration scripts in `sa/db/`
- **Feature flags**: New functionality should be gated behind feature flags for gradual rollout
- **Security first**: All changes must consider security implications and follow Boulder's threat model
- **No AI-specific comments**: Do not add comments to code or diagrams explaining AI actions or modifications; commit messages and PR descriptions are the appropriate place for such explanations

### AI Agent Assistance Areas

AI agents can effectively assist with:

- **ACME endpoint implementation**: Adding new ACME protocol features to WFE2
- **Validation logic**: Implementing new challenge types or validation methods in VA
- **Certificate policies**: Adding policy enforcement in RA and CA services
- **Database operations**: Extending SA with new data models or queries
- **Monitoring and observability**: Adding metrics, logging, and health checks
- **Testing and integration**: Creating unit tests, integration tests, and test fixtures

## Quick Task Routing Guide

Use this decision matrix to quickly identify which services to modify for common development tasks:

| Task Type | Primary Service | Secondary Services | Key Files | Configuration |
|-----------|----------------|-------------------|-----------|---------------|
| **New ACME endpoint** | WFE2 | RA, SA | `wfe2/wfe.go`, `ra/ra.go` | `test/config/wfe2.json` |
| **New challenge type** | VA | RA, WFE2 | `va/va.go`, `core/challenges.go`, `ra/ra.go` | `test/config/va.json` |
| **Certificate policy** | RA, CA | SA | `ra/ra.go`, `ca/ca.go`, `issuance/cert.go` | `test/config/ra.json`, `test/config/ca.json` |
| **Rate limiting** | WFE2, RA | SA | `ratelimits/transaction.go`, `ra/ra.go` | `test/config/wfe2-ratelimit-*.yml` |
| **Database schema** | SA | All services | `sa/model.go`, `sa/db/boulder_sa/*.sql` | `test/config/sa.json` |
| **Domain validation** | VA | RA | `va/dns.go`, `va/http.go`, `va/tlsalpn.go` | `test/config/va.json` |
| **Certificate issuance** | CA | RA, SA, Publisher | `ca/ca.go`, `issuance/cert.go` | `test/config/ca.json` |
| **External integrations** | Publisher | CA, SA | `publisher/publisher.go`, `ctpolicy/ctpolicy.go` | `test/config/publisher.json` |
| **Account management** | WFE2, RA | SA | `wfe2/wfe.go`, `ra/ra.go` | `test/config/wfe2.json` |
| **Monitoring/metrics** | All services | - | Service-specific `*.go` files | Service-specific config files |

**Quick Decision Rules:**
- **Client-facing changes**: Start with WFE2
- **Validation logic**: Start with VA
- **Business rules/policies**: Start with RA
- **Certificate generation**: Start with CA
- **Data persistence**: Start with SA
- **External publishing**: Start with Publisher

## Boulder's Architecture Philosophy

Boulder implements a security-oriented microservices architecture where:

- **Security contexts are separated**: Internet-facing services (WFE2, VA, Publisher) are isolated from internal services (RA, CA, SA)
- **Services communicate via gRPC**: All inter-service communication uses protocol buffers and gRPC for type safety and performance
- **Storage is centralized**: The Storage Authority (SA) acts as the single source of truth for all persistent data
- **Validation is distributed**: Multiple Validation Authority instances provide network perspective diversity for domain validation

## Boulder Services Details

For a detailed breakdown of Boulder's services, their responsibilities, architecture, and interactions, please refer to the [Boulder Services Details](./SERVICES_DETAILS.md) document.

## Development Patterns and Templates

For concrete examples, code templates, and established development patterns used in Boulder, please see the [Development Patterns and Templates](./DEVELOPMENT_PATTERNS.md) document.

## Practical Implementation Guide

For step-by-step instructions on setting up a development environment, implementing new features, and common development workflows, consult the [Practical Implementation Guide](./PRACTICAL_IMPLEMENTATION_GUIDE.md).

### Security Context Separation

**Internet-facing services** (higher risk):

- WFE2 - Public ACME API
- VA - Domain validation
- Publisher - External log submission

**Internal services** (lower risk):

- RA - Internal coordination
- CA - Certificate signing (air-gapped)
- SA - Database operations

### Service Discovery

Boulder uses **Consul** for service discovery and health checking, with configuration in `test/consul/config.hcl`.

## Administrative Tools

Boulder includes various administrative and operational tools in `cmd/`:

- **admin** - Administrative operations
- **cert-checker** - Certificate validation and analysis
- **expiration-mailer** - Certificate expiration notifications
- **log-validator** - Certificate Transparency log validation
- **contact-auditor** - Account contact validation
- **bad-key-revoker** - Compromised key handling

## Development and Testing

**Test Infrastructure:**

- `test/` - Integration and end-to-end tests
- Docker Compose setup for local development
- Extensive unit and integration test coverage
- Load testing and performance validation tools

**Configuration:**

- YAML-based configuration management
- Feature flags for gradual rollouts
- Environment-specific settings
- Rate limiting configuration

**Testing**: Use the provided convenience scripts `t.sh` (standard tests) and `tn.sh` (tests with `config-next`) for running unit and integration tests. For example, using `t.sh`:
    *   Run all tests:
        ```bash
        ./t.sh
        ```
    *   Run unit tests:
        ```bash
        ./t.sh --unit
        ```
    *   Run integration tests:
        ```bash
        ./t.sh --integration
        ```
    *   Run specific tests (note the equals sign):
        ```bash
        ./t.sh --unit --filter=TestDNSAccount01
        ./t.sh --integration --filter=TestSpecificFunction
        ```

**Environment Setup**: If you encounter Docker-related issues during testing:
```bash
docker compose build --pull
```
This resolves most Docker image and dependency issues.

This microservices architecture provides security through isolation, scalability through independent scaling, and maintainability through clear component boundaries. Each service has a well-defined responsibility and communicates through stable gRPC interfaces.

## Troubleshooting Workflow

When encountering issues during Boulder development:

1. **Check documentation first** - Review `*.md` files in boulder-development repo
2. **Environment issues** - Run `docker compose build --pull` for Docker problems
3. **Test failures** - Verify test filter syntax uses equals sign: `--filter=TestName`
4. **Service selection** - Use the Quick Task Routing Guide above to identify correct services
5. **Ask for help** - If documentation doesn't resolve the issue, request assistance rather than proceeding with uncertainty

## Quick Reference Card

**Most Common Development Patterns:**
- New ACME endpoint: Modify WFE2 → RA → SA
- New challenge type: Modify VA → RA → WFE2  
- Certificate policy: Modify RA → CA
- Database changes: Create migration in `sa/db/` → Update SA models
- Testing: Always use `./t.sh` or `./tn.sh`, never direct `go test`
- Docker issues: Run `docker compose build --pull`
