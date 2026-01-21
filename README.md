# SAP Agentic AI Best Practices

Best practices for implementing AI-assisted SAP development using Model Context Protocol (MCP) servers, Kiro-CLI agents, and AWS infrastructure.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [MCP Best Practices for ABAP Accelerator](#2-mcp-best-practices-for-abap-accelerator)
3. [Kiro-CLI Best Practices for Clean Core Agents](#3-kiro-cli-best-practices-for-clean-core-agents)
4. [Security Best Practices](#4-security-best-practices)
5. [Operational Best Practices](#5-operational-best-practices)
6. [AWS Deployment Considerations](#6-aws-deployment-considerations)
7. [Appendices](#7-appendices)

---

## 1. Introduction

### 1.1 Scope

This document provides best practices for organizations implementing AI-assisted SAP development workflows. It covers architectural patterns, agent design, security, and operations without duplicating setup instructions available in referenced documentation.

### 1.2 Target Audience

- Platform engineers deploying SAP MCP infrastructure
- Developers building AI agents for SAP systems
- Security architects reviewing SAP AI integration patterns
- Operations teams maintaining SAP MCP deployments

### 1.3 Component Overview

| Component | Purpose | Documentation |
|-----------|---------|---------------|
| SAP MCP Server | SAP system connectivity via containerized runtime | [SAP ABAP Accelerator Setup Guide](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer) |
| AWS MCP Servers | AWS service integration (64 servers across 8 domains) | [AWS Labs MCP Servers](https://awslabs.github.io/mcp/) |
| Kiro-CLI | AI agent framework for custom workflows | [Kiro CLI Documentation](https://kiro.dev/docs/cli/) |
| Custom Agents | Task-specific AI agents (ATC checks, documentation, etc.) | [Kiro Custom Agents](https://kiro.dev/docs/cli/custom-agents/) |
| AWS Infrastructure | Cloud deployment patterns for MCP servers | [AWS MCP Deployment Guidance](https://aws.amazon.com/solutions/guidance/deploying-model-context-protocol-servers-on-aws/) |

### 1.4 Use Cases

The practices in this document apply to SAP AI workflows such as:

| Use Case | Description |
|----------|-------------|
| Clean Core Assessment | Automated ATC checks with compliance classification (A/B/C/D levels) |
| Custom Code Documentation | AI-generated documentation for Z/Y objects |
| Custom Code Migration Analysis | S/4HANA readiness assessment using migration analysis data |
| Unit Test Development | Create or update ABAP unit test classes |
| Syntax Validation | ABAP syntax checking before activation |
| Unused Code Discovery | Runtime analysis using SUSG statistics |
| Code Search and Discovery | Search and list objects across packages |
| Object Activation | Activate ABAP objects after modifications |

---

## 2. MCP Best Practices for ABAP Accelerator

For MCP architecture concepts, see [MCP Architecture Overview](https://modelcontextprotocol.io/docs/concepts/architecture).

### 2.1 Container Deployment

**Principle**: Deploy MCP servers as containerized applications with explicit resource constraints.

Key practices:

- Use specific image version tags; avoid `latest`
- Apply memory and CPU limits appropriate to workload
- Use `--rm` flag to prevent container accumulation
- Mount secrets as read-only

For container setup instructions, see [SAP ABAP Accelerator Setup Guide](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer).

### 2.2 Multi-System Architecture

**Principle**: Use separate MCP server instances per SAP system.

When connecting to multiple SAP systems (DEV, SBX, etc.), deploy separate container instances rather than sharing a single connection. This provides:

- Clear credential separation
- Independent scaling
- Isolated failure domains
- Per-system audit trails

For multi-system configuration, see [Multiple SAP Systems Configuration](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer#multiple-sap-systems-configuration).

---

## 3. Kiro-CLI Best Practices for Clean Core Agents

### 3.1 Three-Phase Workflow Pattern

**Principle**: Structure agent workflows in three distinct phases with clear gates.

```
Phase 1: Initialization
- Check for existing progress state
- Verify SAP connection
- Discover target objects
- Create progress tracking file
- Gate: Verify all prerequisites before proceeding

Phase 2: Processing
- Process objects in batches
- Checkpoint progress after each batch
- Handle individual failures without stopping
- Output progress indicators

Phase 3: Completion
- Generate summary report
- Archive progress files with timestamp
- Report final statistics
```

This pattern enables crash recovery, progress visibility, and clean separation of concerns.

For agent configuration, see [Kiro Custom Agents Documentation](https://kiro.dev/docs/cli/custom-agents/).

---

## 4. Security Best Practices

### 4.1 Environment Restrictions

**Principle**: Restrict AI-assisted operations to non-production systems.

| Environment | AI Access | Rationale |
|-------------|-----------|-----------|
| Development (DEV) | Permitted | Safe for experimentation |
| Sandbox (SBX) | Permitted | Isolated testing |
| Training (TRN) | Permitted | Educational use |
| Quality (QAS) | Not Permitted | Pre-production data |
| Production (PRD) | Not Permitted | Business critical |

This restriction exists because AI agents can read source code, execute ATC checks, and generate documentation. These operations should not occur against production systems or data.

### 4.2 Credential Management

**Principle**: Never store credentials in configuration files, environment variables, or version control.

#### Recommended Patterns

| Deployment | Credential Storage | Reference |
|------------|-------------------|-----------|
| Local Development | Docker secrets with bind mounts | [SAP ABAP Accelerator - Password Configuration](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer#step-3-configure-sap-password) |
| AWS Cloud | AWS Secrets Manager or Cognito | [AWS MCP Guidance - Security](https://aws.amazon.com/solutions/guidance/deploying-model-context-protocol-servers-on-aws/) |

#### Key Practices

- Store passwords in dedicated secret files with restricted permissions (600)
- Mount secrets as read-only into containers
- Use secure password entry methods that avoid shell history
- Rotate credentials on a defined schedule
- Never commit secrets directories to version control

### 4.3 Network Security

**Principle**: Minimize network exposure of MCP servers.

#### Transport Selection

| Transport | Use Case | Exposure |
|-----------|----------|----------|
| stdio | Local agent deployments | None (stdin/stdout only) |
| HTTP | Remote or multi-client | Requires authentication and TLS |

For local deployments using Kiro-CLI, stdio transport is preferred. The MCP server runs as a subprocess with no network ports exposed.

For HTTP transport (multi-client scenarios), implement:
- Origin header validation
- Localhost binding (127.0.0.1, not 0.0.0.0)
- TLS encryption
- Authentication (OAuth 2.0 recommended for cloud deployments)

### 4.4 Input Validation

**Principle**: Validate all inputs before SAP system interaction.

MCP servers should validate:
- Object names match expected patterns (Z*, Y* for custom code)
- Package names exist and are accessible
- Parameters are free from injection attempts
- Request rates are within acceptable limits

### 4.5 Audit and Logging

**Principle**: Maintain audit trails for all SAP interactions.

Log entries should capture:
- Timestamp
- User or agent identity
- Tool invoked
- SAP object accessed
- Operation result (success/failure)

For cloud deployments, centralize logs using AWS CloudWatch or equivalent. See [AWS MCP Guidance](https://aws.amazon.com/solutions/guidance/deploying-model-context-protocol-servers-on-aws/) for CloudWatch integration patterns.

---

## 5. Operational Best Practices

### 5.1 Health Monitoring

**Principle**: Verify system health before and during batch operations.

Health checks should confirm:
- Container runtime is available
- MCP image is loaded
- Configuration files exist
- SAP system is reachable
- Credentials are valid

For local deployments, run verification commands before starting agents. For cloud deployments, integrate health checks with load balancer configuration.

### 5.2 Resource Management

**Principle**: Clean up resources proactively.

- Use `--rm` flag for automatic container cleanup
- Run periodic system prune to remove unused images
- Archive completed progress files rather than accumulating
- Monitor disk usage for output directories

### 5.3 Output Organization

**Principle**: Maintain consistent directory structure for outputs.

Recommended structure:
```
/reports/
  /atc/           # ATC check results
  /docs/          # Generated documentation
  /unused/        # Unused code analysis
  /executive/     # Business summaries
  /archive/       # Historical progress files
```

Each agent should write to its designated directory. Archive progress files with timestamps after completion for audit purposes.

### 5.4 Timeout Configuration

**Principle**: Set appropriate timeouts for SAP operations.

SAP operations can be slow, particularly:
- ATC checks on large objects or packages
- Source retrieval for complex programs
- Initial connection establishment

Configure MCP client timeout to at least 60 seconds. For batch operations on large packages, consider longer timeouts or implement per-object timeout handling.

### 5.5 Troubleshooting Approach

**Principle**: Follow systematic diagnostic steps.

When issues occur:
1. Verify infrastructure prerequisites (Docker running, image loaded)
2. Test SAP connectivity independently
3. Check credential validity
4. Review MCP server logs
5. Test with minimal scope (single object) before batch operations

For detailed troubleshooting, see [SAP ABAP Accelerator Troubleshooting](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer#error-handling--troubleshooting).

### 5.6 Understanding MCP Error Responses

MCP servers return two types of errors:

| Error Type | Meaning | Action |
|------------|---------|--------|
| Protocol Error | Request itself is invalid (wrong tool name, bad parameters) | Check MCP configuration and tool syntax |
| Execution Error | Request valid but operation failed (object locked, not found) | Check SAP system state and permissions |

**Common execution errors and resolution:**

| Error Message | Cause | Resolution |
|---------------|-------|------------|
| Object not found | Object does not exist or wrong name | Verify object name in SAP |
| Object locked | Another user editing | Wait or contact user |
| Authorization failed | Missing SAP permissions | Check user authorizations |
| Connection timeout | SAP system slow or unreachable | Verify network and SAP availability |

---

## 6. AWS Deployment Considerations

### 6.1 When to Use Cloud Deployment

Local deployment (Docker + Kiro-CLI) is appropriate for:
- Individual developer workstations
- Small-scale assessments
- Development and testing of agents

Cloud deployment (AWS ECS/Fargate) is appropriate for:
- Multi-user environments
- Production-grade operations
- Integration with enterprise security controls
- Centralized monitoring and logging

### 6.2 Key AWS Patterns

For cloud deployment of MCP servers, AWS provides reference architecture covering:

| Pattern | Purpose |
|---------|---------|
| ECS/Fargate | Container orchestration without server management |
| Cognito | OAuth 2.0 authentication for MCP endpoints |
| WAF | Rate limiting and DDoS protection |
| CloudFront | HTTPS termination and edge caching |
| CloudWatch | Centralized logging and monitoring |
| Multi-AZ | High availability across failure domains |

For implementation details, see [AWS Guidance for Deploying MCP Servers](https://aws.amazon.com/solutions/guidance/deploying-model-context-protocol-servers-on-aws/).

### 6.3 SAP-Specific Considerations

When deploying SAP MCP servers on AWS:

- **Network Connectivity**: Ensure ECS tasks can reach SAP systems (VPC peering, Direct Connect, or VPN)
- **Credential Storage**: Use AWS Secrets Manager for SAP passwords rather than file-based secrets
- **Logging**: Include SAP system identifiers in CloudWatch log entries for filtering
- **Scaling**: SAP RFC connections are typically single-threaded; scale horizontally with multiple container instances

---

## 7. Appendices

### Appendix A: Clean Core Level Reference

| Level | Name | Description | Action Required |
|-------|------|-------------|-----------------|
| A | Fully Clean | Uses only released, stable APIs | None - cloud ready |
| B | Pragmatically Clean | Minor findings, informational only | Review and document |
| C | Conditionally Clean | Uses APIs with planned successors | Plan migration path |
| D | Not Clean | Uses deprecated or internal APIs | Remediation required |

### Appendix B: MCP Tool Reference

Standard tools provided by SAP ABAP Accelerator:

**Object Management**

| Tool | Purpose |
|------|---------|
| `aws_abap_cb_get_objects` | List ABAP objects in a package |
| `aws_abap_cb_create_object` | Create new ABAP objects |
| `aws_abap_cb_get_source` | Retrieve source code |
| `aws_abap_cb_update_source` | Update source code |
| `aws_abap_cb_search_object` | Search for objects |

**Development Tools**

| Tool | Purpose |
|------|---------|
| `aws_abap_cb_check_syntax` | ABAP syntax validation |
| `aws_abap_cb_activate_object` | Activate ABAP objects |
| `aws_abap_cb_run_unit_tests` | Execute ABAP unit tests |
| `aws_abap_cb_run_atc_check` | Run ATC checks (supports custom variants) |

**Advanced Features**

| Tool | Purpose |
|------|---------|
| `aws_abap_cb_generate_documentation` | Generate documentation in SAP system |
| `aws_abap_cb_get_migration_analysis` | Custom Code Migration analysis |
| `aws_abap_cb_create_or_update_test_class` | Create or update unit test classes |

For complete tool documentation, see [SAP ABAP Accelerator](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer#available-mcp-tools).

### Appendix C: Troubleshooting Checklist

**Pre-flight Checks**

- [ ] Container runtime is running
- [ ] MCP image is loaded and correct version
- [ ] Configuration file exists with valid syntax
- [ ] Secrets file exists with correct permissions
- [ ] SAP system is network-reachable
- [ ] SAP user credentials are valid and not locked
- [ ] Target environment is non-production

**Common Issues**

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Connection timeout | Network or SAP availability | Verify SAP reachability |
| Authentication failed | Invalid credentials | Check password file content |
| File not found | Path configuration | Use absolute paths |
| Permission denied | File permissions | Set 600 for secrets, 755 for directories |

For detailed troubleshooting, see [SAP ABAP Accelerator Troubleshooting](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer#error-handling--troubleshooting).

---

## References

- [SAP ABAP Accelerator for Amazon Q Developer](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer)
- [AWS Labs MCP Servers](https://awslabs.github.io/mcp/)
- [Kiro CLI Documentation](https://kiro.dev/docs/cli/)
- [Kiro Custom Agents](https://kiro.dev/docs/cli/custom-agents/)
- [AWS Guidance for Deploying MCP Servers](https://aws.amazon.com/solutions/guidance/deploying-model-context-protocol-servers-on-aws/)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [SAP Clean Core](https://www.sap.com/products/erp/s4hana-clean-core.html)

---

*Document Version: 1.0*
