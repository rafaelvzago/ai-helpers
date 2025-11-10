# Security Plugin

CVE analysis and container image vulnerability scanning for Red Hat ecosystem using official Red Hat APIs.

## Overview

The Security plugin provides automated vulnerability assessment capabilities directly within the Claude Code environment. It integrates with Red Hat's official security data sources to provide instant vulnerability analysis, eliminating the need to leave your IDE for security research.

**Key capabilities:**
- **Container vulnerability scanning** using Red Hat Pyxis API
- **CVE information lookup** using Red Hat Security Data API
- **Multi-architecture support** (amd64, arm64, ppc64le, s390x)
- **Security-driven decision making** with grade-based recommendations
- **Corporate environment compatibility** with SSL and proxy support

This plugin addresses critical developer workflow gaps by bringing security assessment into the development process, enabling proactive vulnerability management and security-informed architectural decisions.

## Commands

### `/security:check-image-grade`

Query Red Hat Pyxis API to check container image vulnerability grades and security posture.

**Usage:**
```
/security:check-image-grade <image>
```

**Examples:**
```bash
# Check Red Hat UBI base image
/security:check-image-grade registry.redhat.io/ubi8/ubi:latest

# Analyze OpenShift component security
/security:check-image-grade registry.redhat.io/openshift4/ose-kube-apiserver:v4.14

# Check specific tagged version
/security:check-image-grade registry.redhat.io/ubi9/ubi:9.3
```

**Features:**
- **Health grades**: A-F scale vulnerability assessment
- **CVE breakdown**: Count by severity (Critical, Important, Moderate, Low)
- **Multi-architecture analysis**: Grade consistency across architectures
- **Remediation recommendations**: Actionable next steps based on findings
- **Corporate authentication**: Kerberos keytab integration

**Prerequisites:**
- Red Hat Pyxis API access (requires keytab authentication)
- Kerberos tools (kinit, klist)
- Network access to pyxis.engineering.redhat.com

### `/security:search-cve`

Search and analyze CVE details using the Red Hat Security Data API.

**Usage:**
```
/security:search-cve <CVE-ID>
```

**Examples:**
```bash
# Analyze recent critical vulnerability
/security:search-cve CVE-2024-3400

# Research kernel security issue
/security:search-cve CVE-2023-4623

# Check container runtime vulnerability
/security:search-cve CVE-2024-21626

# Legacy vulnerability analysis
/security:search-cve CVE-2014-0160
```

**Features:**
- **CVSS scoring**: Both v2 and v3 scores with risk interpretation
- **Affected products**: Red Hat product impact analysis
- **Security advisories**: Related RHSA, RHBA, RHEA references
- **Timeline analysis**: Publication, modification, and fix dates
- **Remediation guidance**: Patch availability and update recommendations

**Prerequisites:**
- Network access to access.redhat.com (no authentication required)
- Internet connectivity for NVD cross-references

## Installation

### From ai-helpers Marketplace

```bash
# Add the ai-helpers marketplace
/plugin marketplace add openshift-eng/ai-helpers

# Install security plugin
/plugin install security@ai-helpers

# Verify installation
/security:search-cve --help
```

### Manual Installation (Development)

```bash
# Clone ai-helpers repository
git clone https://github.com/openshift-eng/ai-helpers.git
cd ai-helpers

# Install in Claude Code
mkdir -p ~/.claude/commands
ln -s $(pwd) ~/.claude/commands/ai-helpers

# Restart Claude Code to load plugins
```

## Authentication Setup

### Pyxis API Authentication

The `/security:check-image-grade` command requires Red Hat Pyxis API access via Kerberos authentication.

**Required setup:**

1. **Obtain Service Account Keytab**
   - Request keytab from Red Hat IT or your manager
   - Keytab must have Pyxis API access permissions
   - Default location: `~/ossm-report-sa.keytab`

2. **Configure Keytab Path** (optional)
   ```bash
   export KEYTAB_PATH="/path/to/your/keytab"
   ```

3. **Test Authentication**
   ```bash
   kinit -kt ~/ossm-report-sa.keytab your-principal@REDHAT.COM
   klist  # Verify ticket
   ```

4. **Corporate SSL Configuration** (if needed)
   ```bash
   export VERIFY_SSL=false  # For corporate environments with self-signed certs
   ```

**Troubleshooting:**
- Verify keytab permissions: `ls -la ~/ossm-report-sa.keytab` (should be 600)
- Test Pyxis connectivity: `curl -k https://pyxis.engineering.redhat.com/v1/`
- Contact Red Hat IT for keytab issues or API access problems

## Integration Patterns

### Security Review Workflow

```bash
# 1. Check base image security before using in Dockerfile
/security:check-image-grade registry.redhat.io/ubi8/ubi:latest

# 2. Research specific CVEs mentioned in security scans
/security:search-cve CVE-2024-1234

# 3. Validate security posture of application images
/security:check-image-grade registry.redhat.io/myapp/api:v1.2.3
```

### Incident Response

```bash
# Rapidly assess impact of newly disclosed CVE
/security:search-cve CVE-2024-XXXX

# Check if container images are affected
/security:check-image-grade affected-image:latest
```

### Security-Driven Architecture Decisions

```bash
# Compare base image options for new projects
/security:check-image-grade registry.redhat.io/ubi8/ubi:latest
/security:check-image-grade registry.redhat.io/ubi9/ubi:latest

# Validate security improvements after updates
/security:check-image-grade myapp:old-version
/security:check-image-grade myapp:new-version
```

## Skills Reference

### Pyxis Integration Skill

**Location:** `skills/pyxis-integration/SKILL.md`

Comprehensive implementation guidance for:
- Kerberos authentication patterns (adapted from OSSM Container Grade Reporter)
- Pyxis API client implementation
- Multi-architecture data handling
- Corporate SSL configuration
- Error handling and recovery

**Use cases:**
- Implementing custom Pyxis integrations
- Troubleshooting authentication issues
- Extending vulnerability analysis capabilities
- Building batch processing workflows

## Corporate Environment Support

### Red Hat Internal Networks

- **SSL Configuration**: Supports `VERIFY_SSL=false` for corporate environments
- **Proxy Support**: Compatible with corporate proxy configurations
- **Authentication**: Integrated with Red Hat Kerberos infrastructure
- **API Endpoints**: Uses official Red Hat internal APIs (Pyxis, Security Data)

### Security Considerations

- **Credential Security**: Keytab files stored with appropriate permissions (600)
- **No Hardcoded Credentials**: All authentication via secure keytab files
- **Audit Logging**: API access logged for security compliance
- **Rate Limiting**: Respectful API usage to avoid service disruption

## Contributing

### Adding New Commands

Follow the ai-helpers command definition format:

1. Create command file: `commands/new-command.md`
2. Include required sections: Name, Synopsis, Description, Implementation
3. Add examples and comprehensive argument documentation
4. Test with `make lint` to validate structure

### Extending Capabilities

Potential enhancements:
- **RHACS integration**: Red Hat Advanced Cluster Security scanning
- **Policy enforcement**: Custom security policy validation
- **Historical analysis**: Vulnerability trend tracking
- **Batch processing**: Multiple image analysis workflows
- **SBOM integration**: Software Bill of Materials analysis

### Development Guidelines

- Follow existing code patterns and error handling
- Leverage OSSM Container Grade Reporter patterns for Pyxis integration
- Maintain compatibility with corporate environments
- Include comprehensive documentation and examples
- Test authentication workflows thoroughly

## Support

- **Issues**: https://github.com/openshift-eng/ai-helpers/issues
- **Documentation**: ai-helpers repository README and AGENTS.md
- **Authentication**: Contact Red Hat IT for keytab and API access issues
- **API Questions**: Refer to Red Hat internal documentation for Pyxis and Security Data APIs

## License

This plugin is part of the ai-helpers repository and follows the same licensing terms. See the main repository LICENSE file for details.