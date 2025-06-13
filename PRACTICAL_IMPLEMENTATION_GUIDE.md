## Practical Implementation Guide

This section provides step-by-step instructions for setting up a development environment and implementing new features in Boulder, based on the actual development workflow used by Boulder contributors.

### Development Environment Setup

Boulder uses Docker Compose for local development. All dependencies (MariaDB, Redis, Consul) are containerized, making setup straightforward.

#### Prerequisites

- **Boulder Repository**: This guide assumes you are operating within the root of an already cloned Boulder repository.
  - **If you are already in the Boulder directory, you can skip the following clone and cd steps.**
  - If you need to clone it for the first time, use:
    ```bash
    git clone https://github.com/letsencrypt/boulder/
    cd boulder
    ```
- **Docker Engine 1.13.0+** and **Docker Compose 1.10.0+**
- **At least 2GB RAM** available for Docker
- **Git**: It is recommended to enable `fsckObjects` (e.g., `transfer.fsckObjects = true` in your Git config) for enhanced integrity checks when fetching. This is not enabled by default in Git.

#### Initial Setup (within the Boulder repository directory)

The primary steps to get the development environment running are:

```bash
# Ensure you are in the root of the Boulder repository
# If you are already here, you do not need to change directories.

# Build Docker images and start all services
# This command also handles initial Go compilation if images are being built.
docker compose up

# In a separate terminal (also from the Boulder root directory), run the full test suite.
# This script handles Go compilation inside the Docker environment for subsequent runs.
./t.sh

# Or run with next-generation config:
./tn.sh
```

#### Primary Testing Tools

Boulder provides two main test runners that handle Docker Compose orchestration:

- **`./t.sh`** - Uses `test/config/` (current configuration)
- **`./tn.sh`** - Uses `test/config-next/` (next-generation configuration)

Common test commands:

```bash
# Run all tests (lints, unit, integration)
./t.sh

# Run only unit tests
./t.sh --unit

# Run specific unit tests
./t.sh --unit --filter=./va

# Run integration tests
./t.sh --integration

# Run specific integration tests
./t.sh --filter=TestAkamaiPurgerDrainQueueFails

# List available integration tests
./t.sh --list-integration-tests

# Next-generation config equivalents
./tn.sh --unit
./tn.sh --integration
```

#### Test Filtering Usage

To run a specific test or subset of tests, use the `--filter=...` flag with `./t.sh`. For example:

```bash
./t.sh --integration --filter=TestAkamaiPurgerDrainQueueFails
```

> **Note:** Do not combine `--filter=...` with both `--unit` and `--integration` flags at the same time. Use it with only one specified test type.

#### Troubleshooting Linting and Test Failures

If you encounter unexpected linting errors or test failures when running `./t.sh` or `./tn.sh`, it may be due to an outdated Docker image for Boulder tools. In particular, the linter and other tools are run inside the `boulder-tools` image, which may not always be rebuilt automatically when dependencies change.

**Recommended Fix:**

```bash
# Rebuild the Boulder Docker images, ensuring you have the latest base images and tools
# This often resolves mysterious linting or test failures

docker compose build --pull
```

> **Note:** Running `docker compose build --pull` will build the most up-to-date `boulder-tools` image, which is required for linting and testing to work correctly. If you see linting errors that do not make sense or are not present in the code, always try this command first.

After rebuilding, re-run your test command:

```bash
./t.sh
```

This will ensure you are using the most up-to-date linter and build environment. If you continue to see linting errors, review the output for actionable issues. Some warnings may be present in test code or due to deprecated APIs, and not all are blockers for development.

#### Service Architecture in Docker

The Docker setup includes:

- **boulder**: Main container running all Boulder services
- **bmysql**: MariaDB database
- **bproxysql**: ProxySQL for database load balancing
- **bredis_1-4**: Redis instances for rate limiting
- **bconsul**: Service discovery
- **bjaeger**: Distributed tracing
- **bpkimetal**: Prometheus exporter for PKI-related metrics

Services communicate via:

- **Internal network** (`bouldernet`): 10.77.77.0/24
- **Public networks** (`publicnet`/`publicnet2`): 64.112.117.0/25

#### Configuration Structure

```
test/config/          # Current production-like config (stable)
test/config-next/     # Next-generation config (for testing upcoming changes)
```

Key configuration files:
Key configuration files:

- `wfe2.json` - Web Front End settings
- `ra.json` - Registration Authority settings
- `va.json` - Validation Authority settings
- `wfe2-ratelimit-defaults.yml` - Rate limiting rules
- `wfe2-ratelimit-overrides.yml` - Rate limit exceptions

### Implementing a New DNS Challenge Validation Method

Based on your goal to add a new DNS-based ACME challenge validation method, here's the complete implementation workflow:

#### 1. Pre-Implementation Analysis

First, analyze the existing DNS validation infrastructure:

```bash
# Examine current DNS challenge implementation
./t.sh --unit --filter=./va
grep -r "DNS01" va/
grep -r "validateDNS" va/

# Study existing challenge types
ls va/*dns* va/*challenge*
cat va/dns.go | head -50
```

#### 2. Code Structure for New DNS Challenge

The implementation will span multiple services:

**Validation Authority (VA) - Core Implementation**

- `va/dns.go` - Add new validation logic
- `va/va.go` - Integrate with challenge dispatcher
- `va/proto/va.proto` - Add gRPC method if needed

**Registration Authority (RA) - Orchestration**

- `ra/ra.go` - Add challenge creation logic
- `ra/proto/ra.proto` - Update if new gRPC methods needed

**Web Front End (WFE2) - ACME API**

- `wfe2/wfe.go` - Add new challenge type to directory
- `core/challenges.go` - Define challenge constants

#### 3. Implementation Steps

**Step 1: Define Challenge Type**

```go
// In core/challenges.go
const ChallengeTypeNewDNS = "dns-new" // Replace with actual name

// In core/challenges.go - Add to supported challenges
var ChallengeTypes = map[string]bool{
    ChallengeTypeHTTP01:    true,
    ChallengeTypeDNS01:     true,
    ChallengeTypeTLSALPN01: true,
    ChallengeTypeNewDNS:    true, // Your new challenge
}
```

**Step 2: Implement VA Validation Logic**

```go
// In va/dns.go - Add new validation function
func (va *ValidationAuthorityImpl) validateNewDNS(ctx context.Context, ident identifier.ACMEIdentifier, challenge core.Challenge) error {
    // Implementation following existing DNS validation patterns

    // 1. Construct expected record name and content
    expectedRecord := // Your challenge-specific logic

    // 2. Query DNS with retries and validation
    records, err := va.dnsClient.LookupTXT(ctx, expectedRecord.Name)
    if err != nil {
        return fmt.Errorf("DNS lookup failed: %w", err)
    }

    // 3. Validate record content
    for _, record := range records {
        if record == expectedRecord.Content {
            return nil // Validation successful
        }
    }

    return fmt.Errorf("challenge validation failed")
}
```

**Step 3: Integrate with Challenge Dispatcher**

```go
// In va/va.go - Add to validateChallenge switch statement
func (va *ValidationAuthorityImpl) validateChallenge(ctx context.Context, ident identifier.ACMEIdentifier, challenge core.Challenge) error {
    switch challenge.Type {
    case core.ChallengeTypeHTTP01:
        return va.validateHTTP01(ctx, ident, challenge)
    case core.ChallengeTypeDNS01:
        return va.validateDNS01(ctx, ident, challenge)
    case core.ChallengeTypeTLSALPN01:
        return va.validateTLSALPN01(ctx, ident, challenge)
    case core.ChallengeTypeNewDNS:
        return va.validateNewDNS(ctx, ident, challenge) // Your new method
    default:
        return berrors.UnsupportedError("Unsupported challenge type: %s", challenge.Type)
    }
}
```

**Step 4: Update RA Challenge Creation**

```go
// In ra/ra.go - Add to newAuthorizationPB
func (ra *RegistrationAuthorityImpl) newAuthorizationPB(ident identifier.ACMEIdentifier) (*corepb.Authorization, error) {
    challenges := []core.Challenge{}

    // Existing challenges...

    // Add your new DNS challenge
    if ra.enabledChallenges[core.ChallengeTypeNewDNS] {
        challenges = append(challenges, core.Challenge{
            Type:   core.ChallengeTypeNewDNS,
            Status: core.StatusPending,
            Token:  core.NewToken(),
        })
    }

    // ... rest of function
}
```

**Step 5: Add Configuration Support**

```bash
# Update VA configuration in test/config/va.json and test/config-next/va.json
{
  "va": {
    "enabledChallenges": {
      "http-01": true,
      "dns-01": true,
      "tls-alpn-01": true,
      "dns-new": true  // Enable your new challenge
    }
  }
}
```

#### 4. Testing Implementation

**Unit Tests - Follow VA Patterns**

```go
// In va/dns_test.go
func TestValidateNewDNS(t *testing.T) {
    t.Parallel()

    va, _ := setup(t, "", "")

    testCases := []struct {
        name        string
        identifier  identifier.ACMEIdentifier
        challenge   core.Challenge
        expectError bool
    }{
        {
            name:       "valid challenge",
            identifier: identifier.DNSIdentifier("example.com"),
            challenge:  core.Challenge{Type: core.ChallengeTypeNewDNS, Token: "test-token"},
            expectError: false,
        },
        // Add more test cases...
    }

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            err := va.validateNewDNS(ctx, tc.identifier, tc.challenge)
            if tc.expectError {
                test.AssertError(t, err, "should have failed")
            } else {
                test.AssertNotError(t, err, "should have succeeded")
            }
        })
    }
}
```

**Integration Tests**

```bash
# Run VA-specific tests
./t.sh --unit --filter=./va

# Run full integration tests
./t.sh --integration

# Run specific DNS-related integration tests
./t.sh --filter TestNewDNS
```

#### 5. Feature Flag Integration

To add a feature flag for your new DNS challenge, add a new boolean field to the `Config` struct in `features/features.go`:

```go
// In features/features.go
// Config contains one boolean field for every Boulder feature flag.
type Config struct {
    // ...existing fields...
    NewDNSChallenge bool // Enables the new DNS challenge type
}
```

After adding the field, you can use the flag in your code as follows:

```go
// Use feature flag in VA
if features.Get().NewDNSChallenge {
    // Enable new DNS challenge support
}
```

> **Note:** All feature flags should be added to the `Config` struct in `features/features.go`.

#### 6. Build and Test Workflow

```bash
# Ensure Docker images are up-to-date (especially if Dockerfiles or base images changed)
# docker compose build --pull # Uncomment and run if needed

# Run comprehensive test suite (handles Go compilation within Docker)
./t.sh

# Run with next-generation config
./tn.sh --unit --filter=./va

# Run specific integration tests
./t.sh --integration --filter TestNewDNS

# Check for linting issues
./t.sh --unit | grep -i lint
```

#### 7. Protocol Buffer Updates (if needed)

If adding new gRPC methods:

```bash
# Regenerate protocol buffers after editing .proto files
docker compose run boulder go generate ./...

# Test protocol buffer changes
./t.sh --unit --filter=./va/proto
```

### Database Migrations (if needed)

If your feature requires database changes:

```bash
# Create migration file
touch sa/db/boulder_sa/YYYYMMDD_new_dns_challenge.sql

# Test migration
docker compose run --use-aliases boulder ./sa/migrations.sh -b boulder_sa_test

# Test with new schema
./t.sh --integration
```

### Debugging Tools

```bash
# Check service logs
docker compose logs boulder

# Run single service for debugging
docker compose run boulder go run ./cmd/boulder-va/

# Interactive debugging
docker compose run boulder bash

# Database inspection (general development DB)
docker compose exec bmysql mysql -u root boulder
```

### Common Development Patterns

1. **Start with unit tests**: Implement test cases before core logic
2. **Use existing patterns**: Follow DNS-01 implementation as template
3. **Feature flag**: Gate new functionality behind flags
4. **Configuration**: Update both `test/config/` and `test/config-next/`
5. **Error handling**: Use Boulder's error patterns with proper context
6. **Integration**: Test end-to-end ACME flow with new challenge

This workflow ensures your new DNS challenge validation method integrates properly with Boulder's architecture while following established development patterns.

### Configuration Validation

Boulder uses a custom fork of the [go-playground/validator](https://github.com/letsencrypt/validator) library to validate configurations at startup. When implementing new features that require configuration changes:

- **Review `docs/config-validation.md`** for available validation tags and patterns
- **Use conditional validation tags** (`required_if`, `required_unless`, etc.) to link configuration requirements to feature flags
- **Test validation behavior** with both valid and invalid configurations to ensure proper error messages

**Common Validation Patterns:**

- `validate:"required"` - Field cannot be omitted
- `validate:"omitempty,url"` - Field is optional, but must be a URL when present
- `validate:"required_if=Features.SomeFlag true"` - Field is required only when a feature flag is enabled
- `validate:"min=1,dive,required"` - For arrays/slices, requires at least one element, and all elements must be non-empty

### Feature Flags and Configuration

Boulder uses feature flags (`features/features.go`) to control the activation of new functionality. When implementing features that require configuration:

1. **Add the feature flag** to the `Config` struct in `features/features.go`
2. **Use conditional validation** in configuration structs to make fields required only when features are enabled
3. **Check feature flags at runtime** using `features.Get().FeatureName`

**Example Feature-Dependent Configuration:**

```go
// Configuration field required only when feature is enabled
SomeAPIEndpoint string ` + "`validate:"omitempty,url,required_if=Features.NewFeatureEnabled true"`" + `

// In code, check the feature flag before using the configuration
if features.Get().NewFeatureEnabled {
    // Use SomeAPIEndpoint configuration here
}
```
