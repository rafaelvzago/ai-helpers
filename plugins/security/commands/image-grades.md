---
description: Generate container vulnerability grade report using container-grade-reporter
argument-hint: <config.yaml> [--grade <grades>]
---

## Name

security:image-grades

## Synopsis

```
/security:image-grades <config.yaml> [--grade <grades>]
```

## Description

The `security:image-grades` command generates a vulnerability grade report for container images using the Red Hat Pyxis API via the container-grade-reporter tool. It processes a YAML configuration file containing releases and components, fetches vulnerability grades, and presents a formatted report.

This command is useful for:

- **Release vulnerability assessment**: Check grades for all images in a release
- **Multi-architecture analysis**: View grades across different architectures
- **Security monitoring**: Track vulnerability grades over time
- **Compliance reporting**: Generate reports for security reviews

The command uses the container-grade-reporter tool which must be installed and configured using `/security:set-tool-path` or cloned into the workspace.

## Implementation

1. **Validate YAML Configuration File**: Check that the input file exists and is readable

   - Error if file path not provided
   - Error if file does not exist
   - Verify file is readable

2. **Locate Container Grade Reporter Tool**: Find the tool installation

   **Priority order:**
   - Check saved path in `~/.config/ai-helpers/security-config.json`
   - Check `./container-grade-reporter/` in current directory
   - Check `../container-grade-reporter/` in parent directory

   **If not found:**
   - Display error: "Container Grade Reporter not found. Please run /security:set-tool-path <path> or clone the repository into your workspace"
   - Provide clone command: `git clone https://gitlab.cee.redhat.com/istio/servicemesh-qe/container-grade-reporter.git`

3. **Verify Prerequisites**: Check for required tools and configuration

   - Verify keytab file exists at `~/ossm-report-sa.keytab` or `$KEYTAB_PATH`
   - If keytab not found, provide setup instructions
   - Verify Makefile exists in tool directory

4. **Prepare Output Location**: Create temporary output directory

   - Use `/tmp/security-image-grades-<timestamp>/` for output
   - Ensure directory is created with appropriate permissions

5. **Execute Container Grade Reporter**: Run the tool using Makefile

   ```bash
   cd <tool-path>
   make run-output OUTPUT=<absolute-output-path>/grades.json CONFIG=<absolute-config-path>
   ```

   **Note:** The Makefile automatically handles:
   - Python version detection (3.12+ or fallback to available Python 3.x)
   - Virtual environment setup
   - Dependency installation
   - Kerberos authentication with keytab

   Display progress messages while tool runs (this may take several minutes for multiple repositories)

6. **Parse JSON Output**: Read and parse the generated JSON file

   - Read `grades.json` from output directory
   - Parse JSON structure
   - Handle malformed JSON gracefully

7. **Apply Grade Filter** (if `--grade` argument provided): Filter results by specified grades

   - Check if `--grade` argument was provided
   - Parse grade values: split by comma, normalize to uppercase (e.g., "b,c" → ["B", "C"])
   - Validate each grade is in the valid set: A, B, C, D, F
   - If invalid grade found, display error and list valid options
   - Filter JSON data to only include entries where `grade` matches the filter
   - If no matches found after filtering, display informative message
   - Store filter information for report formatting

8. **Format Vulnerability Report**: Convert JSON to readable text format

   **Output format depends on whether grade filter is active:**

   **Without filter (default):**

   **For each release in the configuration:**
   - Display release name as header
   - For each repository:
     - Repository name
     - For each tag:
       - Tag version
       - Vulnerability grade (A-F) with color indicator
       - Latest patch version
       - Architecture support (grouped if all architectures have same grade/version)
       - End date

   **Summary section:**
   - Total repositories processed
   - Grade distribution (count by grade: A, B, C, D, F)
   - Highlight any critical grades (D, F)

   **With filter (when `--grade` is specified):**

   Display simplified summary listing only matching images:
   - Header showing filtered grade(s)
   - For each matching image:
     - Format: `repository:tag → Grade: X | version | architectures`
   - Group entries by grade for readability
   - Summary: Count of images per grade

9. **Display Results**: Present formatted report to user

   - Use clear visual separators
   - Color-code grades: A/B=✅, C=⚠️, D/F=❌
   - Include actionable summary

10. **Cleanup**: Remove temporary files but preserve JSON for reference

   - Keep JSON output for debugging: inform user of location
   - Remove any temporary copied configuration files

## Return Value

- **Format**: Formatted text report with vulnerability grades

**Example Output:**
```
🛡️ Container Vulnerability Report
Generated: 2025-11-10 15:45:23

Release: OSSM 3.1
════════════════════════════════════════

openshift-service-mesh/istio-cni-rhel9
  1.26 → Grade: B ✅ | v1.26.5 | amd64,arm64,ppc64le,s390x | End: 2025-08-22

openshift-service-mesh/istio-pilot-rhel9
  1.26 → Grade: B ✅ | v1.26.4 | amd64,arm64,ppc64le,s390x | End: 2025-08-20

openshift-service-mesh/kiali-rhel9
  v2.11 → Grade: C ⚠️ | v2.11.3 | amd64,arm64 | End: 2025-06-15

Release: OSSM 2.6
════════════════════════════════════════

openshift-service-mesh/pilot-rhel8
  2.6 → Grade: D ❌ | v2.6.8 | amd64,arm64,ppc64le,s390x | End: 2024-12-31

════════════════════════════════════════
Summary:
  Total repositories: 4
  Grade distribution: 2×B, 1×C, 1×D
  ⚠️  Action required: 1 repository with grade D needs immediate attention

JSON output saved: /tmp/security-image-grades-20251110-154523/grades.json
```

**Filtered Output Example (with `--grade b`):**
```
🛡️ Filtered Container Vulnerability Report (Grade: B)
Generated: 2025-11-10 15:45:23

Grade B Images:
════════════════════════════════════════
openshift-service-mesh/istio-cni-rhel9:1.26 → Grade: B | v1.26.5 | amd64,arm64,ppc64le,s390x
openshift-service-mesh/istio-pilot-rhel9:1.26 → Grade: B | v1.26.4 | amd64,arm64,ppc64le,s390x

════════════════════════════════════════
Summary: 2 images with grade B

JSON output saved: /tmp/security-image-grades-20251110-154523/grades.json
```

**Filtered Output Example (with `--grade c,d,f`):**
```
🛡️ Filtered Container Vulnerability Report (Grades: C, D, F)
Generated: 2025-11-10 15:45:23

Grade C Images:
════════════════════════════════════════
openshift-service-mesh/kiali-rhel9:v2.11 → Grade: C | v2.11.3 | amd64,arm64

Grade D Images:
════════════════════════════════════════
openshift-service-mesh/pilot-rhel8:2.6 → Grade: D | v2.6.8 | amd64,arm64,ppc64le,s390x

════════════════════════════════════════
Summary: 2 images (1×C, 1×D)
⚠️  Action required: 1 repository with grade D needs immediate attention

JSON output saved: /tmp/security-image-grades-20251110-154523/grades.json
```

## Examples

1. **Generate report for OSSM releases**:
   ```
   /security:image-grades ~/ossm-releases.yaml
   ```

2. **Generate report with specific configuration**:
   ```
   /security:image-grades ./configs/ossm-3.1.yaml
   ```

3. **Generate report after setting tool path**:
   ```
   /security:set-tool-path ~/Code/container-grade-reporter
   /security:image-grades ./my-releases.yaml
   ```

4. **Filter by specific grade**:
   ```
   /security:image-grades ~/releases.yaml --grade b
   ```

5. **Filter by multiple grades**:
   ```
   /security:image-grades ~/releases.yaml --grade c,d,f
   ```

6. **Check for critical vulnerabilities only**:
   ```
   /security:image-grades ./configs/ossm-3.1.yaml --grade d,f
   ```

## Arguments

- **$1**: YAML configuration file path
  - Required: Yes
  - Format: Path to YAML file with releases/components structure
  - Example: `~/configs/ossm.yaml`, `./releases.yaml`

- **$2**: Grade filter (optional)
  - Required: No
  - Format: `--grade <grades>` where grades is a comma-separated list
  - Valid grades: A, B, C, D, F (case-insensitive)
  - Examples: `--grade b`, `--grade c,d,f`, `--grade D,F`
  - When specified, only images with matching grades are displayed in simplified format
  - When omitted, all grades are shown in detailed format

## Configuration File Format

The YAML configuration file must follow this structure:

```yaml
releases:
  "OSSM 3.1":
    components:
      - repository: openshift-service-mesh/istio-cni-rhel9
        minimal_tag: ["1.26"]
      - repository: openshift-service-mesh/kiali-rhel9
        minimal_tag: ["v2.11"]
        skip_tag: ["v2.10"]

  "OSSM 2.6":
    components:
      - repository: openshift-service-mesh/pilot-rhel8
        minimal_tag: ["2.6"]
```

**Fields:**
- `releases`: Top-level key containing release definitions
- `"Release Name"`: Name of the release (e.g., "OSSM 3.1")
- `components`: List of repositories to check
- `repository`: Full repository path
- `minimal_tag`: List of minimum tags to include (filters out older versions)
- `skip_tag`: (Optional) List of tags to exclude

## Error Handling

**Common errors:**

1. **Missing configuration file**:
   ```
   Error: Configuration file not found: /path/to/config.yaml
   Usage: /security:image-grades <config.yaml>
   ```

2. **Tool not found**:
   ```
   Error: Container Grade Reporter not found.

   Please either:
   1. Set tool path: /security:set-tool-path <path>
   2. Clone into workspace: git clone https://gitlab.cee.redhat.com/istio/servicemesh-qe/container-grade-reporter.git

   Searched locations:
   - Config: ~/.config/ai-helpers/security-config.json (not found)
   - Workspace: ./container-grade-reporter/ (not found)
   - Parent: ../container-grade-reporter/ (not found)
   ```

3. **Missing keytab**:
   ```
   Error: Keytab file not found: ~/ossm-report-sa.keytab

   The container-grade-reporter requires Kerberos authentication.
   Please ensure your keytab file is available at:
   - ~/ossm-report-sa.keytab (default)
   - Or set KEYTAB_PATH environment variable

   Contact Red Hat IT for keytab access.
   ```

4. **Invalid YAML format**:
   ```
   Error: Invalid YAML configuration file.

   The configuration file must follow this structure:
   releases:
     "Release Name":
       components:
         - repository: path/to/repo
           minimal_tag: ["tag"]

   Please check your configuration file syntax.
   ```

5. **Tool execution failure**:
   ```
   Error: Container Grade Reporter execution failed.

   Check the tool output above for details.
   Common issues:
   - Authentication failure (check keytab)
   - Network connectivity to Pyxis API
   - Invalid repository names in configuration

   For manual troubleshooting, run:
   cd <tool-path> && make run
   ```

6. **Invalid grade filter**:
   ```
   Error: Invalid grade specified: 'X'

   Valid grades are: A, B, C, D, F
   Examples:
   - /security:image-grades config.yaml --grade b
   - /security:image-grades config.yaml --grade c,d,f
   ```

7. **No images matching filter**:
   ```
   No images found matching grade filter: B

   The scan completed successfully but no images have grade B.
   Try running without --grade filter to see all results:
   /security:image-grades config.yaml

   JSON output saved: /tmp/security-image-grades-20251110-154523/grades.json
   ```

## Prerequisites

1. **Container Grade Reporter Tool**:
   - Clone: `git clone https://gitlab.cee.redhat.com/istio/servicemesh-qe/container-grade-reporter.git`
   - Configure: `/security:set-tool-path /path/to/container-grade-reporter`

2. **Kerberos Authentication**:
   - Keytab file at `~/ossm-report-sa.keytab` or `$KEYTAB_PATH`
   - Contact Red Hat IT for keytab access

3. **Network Access**:
   - Access to Red Hat Pyxis API (pyxis.engineering.redhat.com)
   - Corporate network or VPN connection may be required

## Notes

- The tool execution may take several minutes depending on the number of repositories
- Progress is shown in real-time during execution
- JSON output is preserved for debugging and further processing
- The Makefile handles all Python setup, dependencies, and authentication automatically
- Multi-architecture data is grouped when all architectures have identical grades and versions


