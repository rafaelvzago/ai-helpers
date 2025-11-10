---
name: Pyxis API Integration
description: Red Hat Pyxis API integration for container vulnerability data and authentication patterns
---

# Pyxis API Integration

This skill provides comprehensive guidance for integrating with the Red Hat Pyxis API to retrieve container image vulnerability data, health grades, and security assessments. It leverages authentication patterns and API client code that can be adapted from the OSSM Container Grade Reporter project.

## When to Use This Skill

Use this skill when you need to:

- Authenticate with Red Hat Pyxis API using Kerberos keytab
- Query container image vulnerability data and health grades
- Retrieve CVE information for specific container images
- Handle multi-architecture container data (amd64, arm64, ppc64le, s390x)
- Parse Pyxis API responses for vulnerability assessment
- Implement corporate SSL configuration and proxy support

## Prerequisites

1. **Kerberos Authentication Setup**

   - Check for keytab file: `ls ~/ossm-report-sa.keytab` or check `$KEYTAB_PATH`
   - Verify Kerberos tools: `which kinit`, `which klist`
   - Test authentication: `kinit -kt ~/ossm-report-sa.keytab your-principal@REDHAT.COM`
   - Verify ticket: `klist`

2. **Network Access and SSL Configuration**

   - Pyxis API endpoint: `https://pyxis.engineering.redhat.com/v1/`
   - Set SSL verification: `export VERIFY_SSL=false` (for corporate environments)
   - Check connectivity: `curl -k https://pyxis.engineering.redhat.com/v1/`

3. **Python Dependencies** (if using Python client)

   - `requests` - HTTP client library
   - `json` - JSON parsing (built-in)
   - `base64` - Keytab encoding (built-in)
   - `subprocess` - Kerberos authentication (built-in)

## Implementation Steps

### Step 1: Authentication Setup

**Kerberos Keytab Authentication:**

```bash
# Check for keytab file
KEYTAB_PATH="${KEYTAB_PATH:-$HOME/ossm-report-sa.keytab}"
if [ ! -f "$KEYTAB_PATH" ]; then
    echo "Error: Keytab file not found at $KEYTAB_PATH"
    echo "Please ensure keytab is available for Pyxis authentication"
    exit 1
fi

# Perform kinit authentication
kinit -kt "$KEYTAB_PATH" "your-service-account@REDHAT.COM"
if [ $? -ne 0 ]; then
    echo "Error: Kerberos authentication failed"
    exit 1
fi

echo "✅ Successfully authenticated with Kerberos"
```

**SSL Configuration:**
```bash
# Configure SSL verification for corporate environments
VERIFY_SSL="${VERIFY_SSL:-true}"
CURL_SSL_OPTS=""
if [ "$VERIFY_SSL" = "false" ]; then
    CURL_SSL_OPTS="-k"
    echo "⚠️  SSL verification disabled for corporate environment"
fi
```

### Step 2: Image Information Query

**Query Image by Repository:**

```bash
# Extract registry, repository, and tag from image reference
parse_image_reference() {
    local image_ref="$1"
    local registry=""
    local repository=""
    local tag="latest"

    # Parse image reference format
    if [[ "$image_ref" == *":"* ]]; then
        tag="${image_ref##*:}"
        image_ref="${image_ref%:*}"
    fi

    if [[ "$image_ref" == *"/"* ]]; then
        registry="${image_ref%%/*}"
        repository="${image_ref#*/}"
    else
        repository="$image_ref"
    fi

    echo "$registry|$repository|$tag"
}

# Query Pyxis for image information
query_image_info() {
    local registry="$1"
    local repository="$2"
    local tag="$3"

    local url="https://pyxis.engineering.redhat.com/v1/repositories/registry/${registry}/repository/${repository}"

    curl $CURL_SSL_OPTS -s --negotiate -u: "$url" | jq '.'
}
```

**Query Specific Image by ID:**

```bash
# Query specific image details
query_image_by_id() {
    local image_id="$1"

    local url="https://pyxis.engineering.redhat.com/v1/images/id/${image_id}"

    curl $CURL_SSL_OPTS -s --negotiate -u: "$url" | jq '.'
}

# Query image vulnerabilities
query_image_vulnerabilities() {
    local image_id="$1"

    local url="https://pyxis.engineering.redhat.com/v1/images/id/${image_id}/vulnerabilities"

    curl $CURL_SSL_OPTS -s --negotiate -u: "$url" | jq '.'
}
```

### Step 3: Health Grade Assessment

**Query Repository Grades:**

```bash
# Get health grades for a repository
query_repository_grades() {
    local registry="$1"
    local repository="$2"

    local url="https://pyxis.engineering.redhat.com/v1/repositories/registry/${registry}/repository/${repository}/grades"

    curl $CURL_SSL_OPTS -s --negotiate -u: "$url" | jq '.'
}

# Parse grade information
parse_grade_data() {
    local grade_data="$1"

    echo "$grade_data" | jq -r '
    {
        "grade": .grade,
        "grade_sources": .grade_sources,
        "cve_counts": .cve_counts,
        "architectures": [.grade_sources[].arch] | unique,
        "worst_grade": .grade_sources | map(.grade) | max,
        "total_cves": (.cve_counts.critical // 0) + (.cve_counts.important // 0) + (.cve_counts.moderate // 0) + (.cve_counts.low // 0)
    }'
}
```

### Step 4: Multi-Architecture Analysis

**Architecture Consistency Check:**

```bash
# Analyze multi-architecture consistency
analyze_architecture_consistency() {
    local grade_data="$1"

    echo "$grade_data" | jq -r '
    .grade_sources as $sources |
    ($sources | group_by(.arch) | map({
        arch: .[0].arch,
        grade: .[0].grade,
        cve_counts: .[0].cve_counts
    })) as $arch_grades |

    {
        "architecture_summary": $arch_grades,
        "consistent_grades": ($arch_grades | map(.grade) | unique | length == 1),
        "worst_architecture": ($arch_grades | max_by(.grade | if . == "A" then 1 elif . == "B" then 2 elif . == "C" then 3 elif . == "D" then 4 else 5 end)),
        "best_architecture": ($arch_grades | min_by(.grade | if . == "A" then 1 elif . == "B" then 2 elif . == "C" then 3 elif . == "D" then 4 else 5 end))
    }'
}
```

### Step 5: Vulnerability Assessment Report

**Generate Comprehensive Report:**

```bash
# Generate vulnerability analysis report
generate_vulnerability_report() {
    local image_ref="$1"
    local grade_data="$2"

    echo "🛡️ Security Analysis: $image_ref"
    echo ""

    # Extract overall grade
    local overall_grade=$(echo "$grade_data" | jq -r '.grade // "Unknown"')

    # Grade display with emoji
    case "$overall_grade" in
        "A") echo "📊 Vulnerability Grade: A (Excellent) ✅" ;;
        "B") echo "📊 Vulnerability Grade: B (Good) ✅" ;;
        "C") echo "📊 Vulnerability Grade: C (Moderate) ⚠️" ;;
        "D") echo "📊 Vulnerability Grade: D (Poor) ❌" ;;
        "F") echo "📊 Vulnerability Grade: F (Critical) 🚨" ;;
        *) echo "📊 Vulnerability Grade: $overall_grade ❓" ;;
    esac

    # CVE breakdown
    echo "├─ Critical CVEs: $(echo "$grade_data" | jq -r '.cve_counts.critical // 0')"
    echo "├─ Important CVEs: $(echo "$grade_data" | jq -r '.cve_counts.important // 0')"
    echo "├─ Moderate CVEs: $(echo "$grade_data" | jq -r '.cve_counts.moderate // 0')"
    echo "└─ Low CVEs: $(echo "$grade_data" | jq -r '.cve_counts.low // 0')"
    echo ""

    # Architecture analysis
    echo "🏗️ Multi-Architecture Support:"
    echo "$grade_data" | jq -r '.grade_sources[] | "├─ \(.arch): Grade \(.grade) \(if .grade == "A" or .grade == "B" then "✅" elif .grade == "C" then "⚠️" else "❌" end)"'
    echo ""

    # Recommendations
    echo "💡 Recommendations:"
    local critical_cves=$(echo "$grade_data" | jq -r '.cve_counts.critical // 0')
    local important_cves=$(echo "$grade_data" | jq -r '.cve_counts.important // 0')

    if [ "$critical_cves" -gt 0 ]; then
        echo "- 🚨 URGENT: Address $critical_cves Critical CVEs immediately"
    fi

    if [ "$important_cves" -gt 0 ]; then
        echo "- ⚠️  Review $important_cves Important CVEs for potential impact"
    fi

    case "$overall_grade" in
        "F") echo "- 🚨 Immediate action required - consider alternative base image" ;;
        "D") echo "- ❌ Schedule security updates as high priority" ;;
        "C") echo "- ⚠️  Plan security updates for next maintenance window" ;;
        "A"|"B") echo "- ✅ Image security posture is acceptable" ;;
    esac

    echo ""
    echo "🔗 Pyxis Details: https://pyxis.engineering.redhat.com/..."
}
```

## Error Handling

### Common Error Scenarios

**Authentication Failures:**
```bash
handle_auth_error() {
    echo "❌ Authentication Error: Unable to authenticate with Pyxis API"
    echo ""
    echo "Troubleshooting steps:"
    echo "1. Verify keytab file exists: ls $KEYTAB_PATH"
    echo "2. Check Kerberos ticket: klist"
    echo "3. Test manual authentication: kinit -kt $KEYTAB_PATH your-principal@REDHAT.COM"
    echo "4. Verify network connectivity to Pyxis API"
    echo ""
    echo "For help, contact Red Hat IT or your system administrator"
}
```

**Image Not Found:**
```bash
handle_image_not_found() {
    local image_ref="$1"
    echo "❌ Image Not Found: $image_ref not found in Pyxis database"
    echo ""
    echo "Possible reasons:"
    echo "1. Image may not be a Red Hat container image"
    echo "2. Image reference may be incorrect"
    echo "3. Image may not be published to Red Hat registries"
    echo ""
    echo "Try searching for similar images or verify the image reference"
}
```

**Network/SSL Issues:**
```bash
handle_network_error() {
    echo "❌ Network Error: Unable to connect to Pyxis API"
    echo ""
    echo "Troubleshooting steps:"
    echo "1. Check network connectivity: ping pyxis.engineering.redhat.com"
    echo "2. Test SSL connection: curl -k https://pyxis.engineering.redhat.com/v1/"
    echo "3. Set SSL verification: export VERIFY_SSL=false (for corporate networks)"
    echo "4. Check proxy configuration if behind corporate firewall"
}
```

## Integration Examples

### Complete Implementation Example

```bash
#!/bin/bash
# Complete Pyxis integration example

check_image_grade() {
    local image_ref="$1"

    # Validate input
    if [ -z "$image_ref" ]; then
        echo "Error: Image reference required"
        echo "Usage: check_image_grade <image>"
        return 1
    fi

    # Parse image reference
    local parsed=$(parse_image_reference "$image_ref")
    local registry=$(echo "$parsed" | cut -d'|' -f1)
    local repository=$(echo "$parsed" | cut -d'|' -f2)
    local tag=$(echo "$parsed" | cut -d'|' -f3)

    echo "🔍 Analyzing: $image_ref"
    echo "📍 Registry: $registry"
    echo "📦 Repository: $repository"
    echo "🏷️  Tag: $tag"
    echo ""

    # Authenticate
    if ! authenticate_kerberos; then
        handle_auth_error
        return 1
    fi

    # Query grades
    local grade_data=$(query_repository_grades "$registry" "$repository")

    if [ $? -ne 0 ] || [ -z "$grade_data" ]; then
        handle_image_not_found "$image_ref"
        return 1
    fi

    # Generate report
    generate_vulnerability_report "$image_ref" "$grade_data"

    return 0
}
```

### Python Integration Alternative

For more complex data processing, you can also implement Pyxis integration in Python, adapting patterns from the OSSM Container Grade Reporter:

```python
# Reference implementation patterns from OSSM project:
# - src/ossm_image_grades/api/pyxis_client.py: API client with authentication
# - src/ossm_image_grades/core/grades.py: Grade processing logic
# - src/ossm_image_grades/config/auth.py: Kerberos authentication patterns
# - Makefile: Authentication automation and keytab handling
```

## Advanced Features

### Batch Processing Multiple Images

```bash
# Process multiple images
check_multiple_images() {
    local images=("$@")

    for image in "${images[@]}"; do
        echo "=================="
        check_image_grade "$image"
        echo ""
    done
}
```

### Grade Trend Analysis

```bash
# Compare current vs previous grades (if historical data available)
compare_grade_history() {
    local image_ref="$1"

    # Implementation would query historical grade data
    # and compare trends over time
    echo "📈 Grade Trend Analysis: (Feature for future enhancement)"
}
```

## Security Considerations

1. **Keytab Security**: Store keytab files securely with appropriate permissions (600)
2. **SSL Verification**: Only disable SSL verification in trusted corporate environments
3. **Credential Storage**: Never hardcode credentials in scripts or commands
4. **Audit Logging**: Log all Pyxis API access for security audit trails
5. **Rate Limiting**: Respect API rate limits to avoid service disruption

## References

- **OSSM Container Grade Reporter**: Reference implementation with working Pyxis integration
- **Pyxis API Documentation**: Red Hat internal documentation for API endpoints
- **Kerberos Authentication**: Red Hat IT guidance for service account setup
- **Corporate SSL Configuration**: Network team guidance for proxy and SSL settings