---
description: Search and analyze CVE details using Red Hat Security Data API
argument-hint: <CVE-ID>
---

## Name

security:search-cve

## Synopsis

```
/security:search-cve <CVE-ID>
```

## Description

The `security:search-cve` command queries the Red Hat Security Data API to retrieve comprehensive information about a specific CVE (Common Vulnerabilities and Exposures). It provides detailed vulnerability analysis without requiring authentication, offering instant CVE reference capabilities within the IDE.

This command is useful for:

- **Rapid CVE analysis** during security reviews and incident response
- **CVSS score interpretation** and risk assessment
- **Affected product identification** across Red Hat portfolio
- **Remediation guidance** and patch availability
- **Security advisory correlation** (RHSA, RHBA, RHEA)
- **Vulnerability timeline analysis** (publication, disclosure, fix dates)

The Red Hat Security Data API provides authoritative information about CVEs affecting Red Hat products, including CVSS scores, affected package versions, and official remediation guidance.

## Implementation

1. **Validate CVE Format**: Verify CVE identifier format and structure

   - CVE format: `CVE-YYYY-NNNN` (e.g., CVE-2024-1234)
   - Validate year range (1999-current)
   - Validate number format and length
   - Provide format correction suggestions for invalid input

2. **Query Red Hat Security Data API**: Retrieve CVE information

   - API endpoint: `https://access.redhat.com/hydra/rest/securitydata/cve/{CVE-ID}.json`
   - No authentication required
   - Handle rate limiting with exponential backoff
   - Support both XML and JSON response formats (prefer JSON)

3. **Parse CVE Data**: Extract key vulnerability information

   - CVE identifier and aliases
   - CVSS v2/v3 scores and vector strings
   - Severity classification (Low, Moderate, Important, Critical)
   - CWE (Common Weakness Enumeration) mapping
   - Publication and modification dates

4. **Analyze Affected Products**: Identify impacted Red Hat products and versions

   - Parse affected product list
   - Extract package names and version ranges
   - Identify fixed versions where available
   - Categorize by product family (RHEL, OpenShift, etc.)

5. **Extract Security Advisories**: Correlate related security advisories

   - Red Hat Security Advisories (RHSA)
   - Red Hat Bug Advisories (RHBA)
   - Red Hat Enhancement Advisories (RHEA)
   - Advisory links and references

6. **Generate Vulnerability Report**: Format comprehensive CVE analysis

   - CVE summary with CVSS scoring
   - Affected products and remediation status
   - Security advisory links
   - Timeline information (disclosure, publication, fix)
   - Remediation recommendations

7. **Error Handling**: Graceful handling of lookup failures

   - CVE not found in Red Hat database
   - Network connectivity issues
   - API rate limiting
   - Invalid CVE format

## Return Value

- **Format**: Detailed CVE analysis report with CVSS scores, affected products, and remediation guidance

**Example Output:**
```
🔍 CVE Analysis: CVE-2024-1234

📊 Risk Assessment:
├─ CVSS v3.1: 8.8 (High)
├─ CVSS v2.0: 7.2 (High)
├─ Severity: Important
└─ CWE: CWE-787 (Out-of-bounds Write)

📅 Timeline:
├─ Published: 2024-03-15
├─ Modified: 2024-03-20
└─ Red Hat Impact: 2024-03-16

🎯 Affected Products:
├─ Red Hat Enterprise Linux 8: Fixed in kernel-4.18.0-553.el8
├─ Red Hat Enterprise Linux 9: Fixed in kernel-5.14.0-427.el9
├─ OpenShift Container Platform 4.14: Under investigation
└─ Red Hat OpenStack Platform 16.2: Not affected

📋 Security Advisories:
├─ RHSA-2024:1234 - Important: kernel security update
└─ RHSA-2024:1235 - Important: kernel-rt security update

💡 Remediation:
- Apply available security updates for affected systems
- Priority: High (CVSS 8.8) - Schedule immediate patching
- No workarounds available - patching required

🔗 References:
- Red Hat CVE Database: https://access.redhat.com/security/cve/CVE-2024-1234
- NVD Reference: https://nvd.nist.gov/vuln/detail/CVE-2024-1234
```

## Examples

1. **Search recent critical CVE**:
   ```
   /security:search-cve CVE-2024-3400
   ```

2. **Analyze kernel vulnerability**:
   ```
   /security:search-cve CVE-2023-4623
   ```

3. **Check OpenSSL CVE**:
   ```
   /security:search-cve CVE-2023-2650
   ```

4. **Research container runtime CVE**:
   ```
   /security:search-cve CVE-2024-21626
   ```

5. **Legacy CVE analysis**:
   ```
   /security:search-cve CVE-2014-0160
   ```

## Arguments

- **$1**: CVE identifier
  - Required: Yes
  - Format: CVE-YYYY-NNNN (e.g., CVE-2024-1234)
  - Validation: Must match standard CVE format
  - Examples: `CVE-2024-3400`, `CVE-2023-4623`, `CVE-2022-0847`