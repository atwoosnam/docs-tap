# Enforce compliance policy using Open Policy Agent

## <a id="writing-pol-temp"></a>Writing a policy template

The Scan Policy custom resource (CR) allows you to define a Rego file for policy enforcement that you can reuse across image scan and source scan CRs.

The Scan Controller supports policy enforcement by using an Open Policy Agent (OPA) engine with Rego files. This allows scan results to be validated for company policy compliance and can prevent source code from being built or images from being deployed.

## <a id="rego-file-contract"></a>Rego file contract

To define a Rego file for an image scan or source scan, you must comply with the requirements defined for every Rego file for the policy verification to work properly.

- **Package main:** The Rego file must define a package in its body called `main`, because the system looks for this package to verify the scan results compliance.

- **Input match:** The Rego file evaluates one vulnerability match at a time, iterating as many times as different vulnerabilities are found in the scan. The match structure is accessed in the `input.currentVulnerability` object inside the Rego file and has the [CycloneDX](https://cyclonedx.org/docs/1.3/) format.

- **deny rule:** The Rego file must define inside its body a `deny` rule. `deny` is a set of error messages that is returned to the user. Each rule you write adds to that set of error messages. If the conditions in the body of the `deny` statement are true then the user is handed an error message. If false, the vulnerability is allowed in the Source or Image scan.

## <a id="define-rego-file"></a>Define a Rego file for policy enforcement

Follow these steps to define a Rego file for policy enforcement that you can reuse across image scan and source scan CRs that output in the CycloneDX XML format.

>**Note** The Snyk Scanner outputs SPDX JSON. See [Sample ScanPolicy for Snyk in SPDX JSON format](install-snyk-integration.md#snyk-scan-policy) for an example of a ScanPolicy formatted for SPDX JSON output.

1. Create a scan policy with a Rego file. Here is a sample scan policy resource:

    ```yaml
    ---
    apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
    kind: ScanPolicy
    metadata:
      name: scan-policy
      labels:
        app.kubernetes.io/part-of: enable-in-gui
    spec:
      regoFile: |
        package main

        # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
        notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]
        ignoreCves := []

        contains(array, elem) = true {
          array[_] = elem
        } else = false { true }

        isSafe(match) {
          severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
          some i
          fails := contains(notAllowedSeverities, severities[i])
          not fails
        }

        isSafe(match) {
          ignore := contains(ignoreCves, match.id)
          ignore
        }

        deny[msg] {
          comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
          some i
          comp := comps[i]
          vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
          some j
          vuln := vulns[j]
          ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
          not isSafe(vuln)
          msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
        }
    ```

    You can edit the following text boxes of the Rego file as part of the [CVE triage workflow](../scst-scan/triaging-and-remediating-cves.hbs.md#amend-scan-policy):

    - `notAllowedSeverities` contains the categories of CVEs that cause the SourceScan or ImageScan failing policy enforcement. The following is an example of how an `app-operator` might decide to only block "Critical", "High" and "UnknownSeverity" CVEs.

      ```yaml
      ...
      spec:
        regoFile: |
          package main

          # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
          notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]
          ignoreCves := []
      ...
      ```

    - `ignoreCves` contains individual ignored CVEs when determining policy enforcement. The following is an example of how an `app-operator` might decide to ignore `CVE-2018-14643` and `GHSA-f2jv-r9rf-7988` if they are false positives. See [A Note on Vulnerability Scanners](overview.hbs.md#scst-scan-note) for more details.

      ```yaml
      ...
      spec:
        regoFile: |
          package main

          notAllowedSeverities := []
          ignoreCves := ["CVE-2018-14643", "GHSA-f2jv-r9rf-7988"]
      ...
      ```

2. Deploy the scan policy to the cluster by running:

    ```console
    kubectl apply -f <path_to_scan_policy>/<scan_policy_filename>.yaml -n <desired_namespace>
    ```

See how scan policies are used in the CVE triage workflow in the [Triaging and Remediating CVEs](../scst-scan/triaging-and-remediating-cves.hbs.md#amend-scan-policy)

## <a id="more-detail"></a>Further refine the Scan Policy for use

The scan policy earlier is provided to demonstrate how vulnerabilities are ignored during a compliance check. It has a lack of auditability as to why a vulnerability is ignored. You might want to allow an exception, where a build with a failing vulnerability is allowed to progress through a supply chain. You can allow this exception for a certain period of time, requiring an expiration date. Vulnerability Exploitability Exchange (VEX) documents are gaining popularity to capture security advisory information pertaining to vulnerabilities. You can express these use cases with Rego.

For example, the following scan policy includes an additional text box to capture comments regarding why a vulnerability is ignored. The `notAllowedSeverities` array remains an array of strings, but the `ignoreCves` array was updated from an array of strings to an array of objects. This causes a change to the `contains` function, splitting it into separate functions for each array.

```yaml
---
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: scan-policy
  labels:
    app.kubernetes.io/part-of: enable-in-gui
spec:
  regoFile: |
    package main

    # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
    notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]

    # List of known vulnerabilities to ignore when deciding whether to fail compliance. Example:
    # ignoreCves := [
    #   {
    #     "id": "CVE-2018-14643",
    #     "detail": "Determined affected code is not in the execution path."
    #   }
    # ]
    ignoreCves := []

    containsSeverity(array, elem) = true {
      array[_] = elem
    } else = false { true }

    isSafe(match) {
      severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
      some i
      fails := containsSeverity(notAllowedSeverities, severities[i])
      not fails
    }

    containsCve(array, elem) = true {
      array[_].id = elem
    } else = false { true }

    isSafe(match) {
      ignore := containsCve(ignoreCves, match.id)
      ignore
    }

    deny[msg] {
      comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
      some i
      comp := comps[i]
      vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
      some j
      vuln := vulns[j]
      ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
      not isSafe(vuln)
      msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
    }
```

Including an expiration text box and only allowing the vulnerability to be ignored for a period of time is shown in the following example:

```yaml
---
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: scan-policy
  labels:
    app.kubernetes.io/part-of: enable-in-gui
spec:
  regoFile: |
    package main

    # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
    notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]

    # List of known vulnerabilities to ignore when deciding whether to fail compliance. Example:
    # ignoreCves := [
    #   {
    #     "id": "CVE-2018-14643",
    #     "detail": "Determined affected code is not in the execution path.",
    #     "expiration": "2022-Dec-31"
    #   }
    # ]
    ignoreCves := []

    containsSeverity(array, elem) = true {
      array[_] = elem
    } else = false { true }

    isSafe(match) {
      severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
      some i
      fails := containsSeverity(notAllowedSeverities, severities[i])
      not fails
    }

    containsCve(array, elem) = true {
      array[_].id = elem
      curr_time := time.now_ns()
      date_format := "2006-Jan-02"
      expire_time := time.parse_ns(date_format, array[_].expiration)
      curr_time < expire_time
    } else = false { true }

    isSafe(match) {
      ignore := containsCve(ignoreCves, match.id)
      ignore
    }

    deny[msg] {
      comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
      some i
      comp := comps[i]
      vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
      some j
      vuln := vulns[j]
      ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
      not isSafe(vuln)
      msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
    }
```

## <a id="gui-view-scan-policy"></a>Enable Tanzu Application Platform GUI to view ScanPolicy Resource

For the Tanzu Application Platform GUI to view the ScanPolicy resource, it must have a matching `kubernetes-label-selector` with a `part-of` prefix.

Here is a portion of a ScanPolicy that is viewable by the Tanzu Application Platform GUI:
```yaml
---
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: scan-policy
  labels:
    app.kubernetes.io/part-of: enable-in-gui
spec:
  regoFile: |
    ...
```
>**Note** The value for the label can be anything. The Tanzu Application Platform GUI is looking for the existence of the `part-of` prefix string and doesn't match for anything else specific.

## <a id="deprecated-rego-file"></a> Deprecated Rego file Definition

Before Scan Controller v1.2.0, you must use the following format where the rego file differences are:

- The package name must be `package policies` instead of `package main`.
- The deny rule is a Boolean `isCompliant` instead of `deny[msg]`.
  - **isCompliant rule:** The Rego file must define inside its body an `isCompliant` rule. This must be a Boolean type containing the result whether the vulnerability violates the security policy or not. If `isCompliant` is `true`, the vulnerability is allowed in the Source or Image scan; `false` is considered otherwise. Any scan that finds at least one vulnerability that evaluates to `isCompliant=false` makes the `PolicySucceeded` condition set to false.
- Here is a sample scan policy resource:
```yaml
apiVersion: scanning.apps.tanzu.vmware.com/v1alpha1
kind: ScanPolicy
metadata:
  name: v1alpha1-scan-policy
  labels:
    app.kubernetes.io/part-of: enable-in-gui
spec:
  regoFile: |
    package policies

    default isCompliant = false

    ignoreSeverities := ["Critical", "High"]

    contains(array, elem) = true {
      array[_] = elem
    } else = false { true }

    isCompliant {
      ignore := contains(ignoreSeverities, input.currentVulnerability.Ratings.Rating[_].Severity)
      ignore
    }
```
