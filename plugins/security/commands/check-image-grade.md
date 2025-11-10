---
description: Check container image vulnerability grade using Red Hat Pyxis API
argument-hint: <image>
---

## Name

security:check-image-grade

## Synopsis

```
/security:check-image-grade <image>
```

## Description

The `security:check-image-grade` command queries the Red Hat Pyxis API to retrieve vulnerability assessment and health grade information for a specified container image. It provides instant security posture analysis without leaving the IDE, leveraging Red Hat's official container vulnerability database.

This command is useful for:

- **Instant vulnerability assessment** of container images during development
- **Security-driven decision making** when selecting base images
- **Multi-architecture compatibility analysis** (amd64, arm64, ppc64le, s390x)
- **CVE count and severity analysis** for risk assessment
- **Grade-based remediation guidance** (A=excellent, F=critical issues)

The vulnerability grading system uses Red Hat's A-F scale where:
- **A-B**: Low risk, production ready
- **C**: Moderate risk, review recommended
- **D**: High risk, address soon
- **F**: Critical risk, immediate attention required

## Implementation

1. **Parse and Validate Image Reference**: Extract and validate the container image reference

   - Support formats: `registry/repository`, `registry/repository:tag`
   - Default to latest tag if none specified
   - Validate image reference format

2. **Authenticate with Pyxis API**: Establish authenticated session using Kerberos

   - Use skill: `plugins/security/skills/pyxis-integration/`
   - Check for keytab file at `~/ossm-report-sa.keytab` or `$KEYTAB_PATH`
   - Perform kinit authentication if keytab available
   - Handle SSL verification based on corporate environment (`VERIFY_SSL`)

3. **Query Image Metadata**: Retrieve image information from Pyxis

   - API endpoint: `https://pyxis.engineering.redhat.com/v1/images/id/{image_id}`
   - Alternative: Repository search if direct ID lookup fails
   - Extract image vulnerabilities and grade information

4. **Query Vulnerability Grade**: Get health grade assessment

   - API endpoint: Repository grades API for latest vulnerability assessment
   - Parse grade information (A, B, C, D, F)
   - Extract CVE count by severity (Critical, Important, Moderate, Low)

5. **Multi-Architecture Analysis**: Check architecture support and consistency

   - Identify available architectures (amd64, arm64, ppc64le, s390x)
   - Compare grades across architectures
   - Flag inconsistencies for review

6. **Generate Analysis Report**: Format comprehensive vulnerability report

   - Overall health grade with color-coded display
   - CVE breakdown by severity level
   - Multi-architecture support matrix
   - Remediation recommendations based on grade
   - Suggested actions (upgrade base image, apply patches, etc.)

7. **Error Handling**: Graceful handling of common failure scenarios

   - Image not found in Pyxis
   - Authentication failures
   - Network connectivity issues
   - API rate limiting

## Return Value

- **Format**: Structured vulnerability report with grade analysis and remediation guidance

**Example Output:**
```
🛡️ Security Analysis: registry.redhat.io/ubi8/ubi:latest

📊 Vulnerability Grade: B (Good)
├─ Critical CVEs: 0
├─ Important CVEs: 2
├─ Moderate CVEs: 8
└─ Low CVEs: 15

🏗️ Multi-Architecture Support:
├─ amd64: Grade B ✅
├─ arm64: Grade B ✅
├─ ppc64le: Grade B ✅
└─ s390x: Grade C ⚠️

💡 Recommendations:
- Review 2 Important CVEs for potential impact
- Consider updating s390x architecture (Grade C)
- Monitor for security updates

🔗 Pyxis Details: https://pyxis.engineering.redhat.com/...
```

## Examples

1. **Check Red Hat UBI image**:
   ```
   /security:check-image-grade registry.redhat.io/ubi8/ubi:latest
   ```

2. **Check RHEL base image**:
   ```
   /security:check-image-grade registry.redhat.io/rhel8/rhel:latest
   ```

3. **Check specific tag**:
   ```
   /security:check-image-grade registry.redhat.io/ubi9/ubi:9.3
   ```

4. **Check OpenShift component**:
   ```
   /security:check-image-grade registry.redhat.io/openshift4/ose-kube-apiserver:v4.14
   ```

## Arguments

- **$1**: Container image reference (registry/repository[:tag])
  - Required: Yes
  - Format: Standard container image reference
  - Examples: `registry.redhat.io/ubi8/ubi`, `registry.redhat.io/ubi8/ubi:latest`