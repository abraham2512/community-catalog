# update-jira-status (Pipeline)

A Tekton `Pipeline` that updates JIRA issues from "Modified" to "On QA" status and adds build information as comments after a successful build.

## Overview

This pipeline is designed to automatically update JIRA issues when builds complete successfully. It:

1. Extracts JIRA issue information from the Release artifacts
2. Checks if issues are in "Modified" status
3. Moves issues from "Modified" to "On QA" status
4. Adds a comment with build information

The pipeline supports authentication to JIRA using credentials stored in Kubernetes secrets and can handle multiple JIRA issues per release.

## Parameters

| Name                    | Description                                                           | Optional | Default value |
|-------------------------|-----------------------------------------------------------------------|----------|---------------|
| `dataPath`              | Path to the JSON string of the merged data to use in the data workspace | No       | -             |
| `buildInfo`             | Information about the build that was generated (e.g., image digest, build ID, etc.) | No       | -             |
| `ociStorage`            | The OCI repository where the Trusted Artifacts are stored | Yes      | "empty"       |
| `ociArtifactExpiresAfter`| Expiration date for the trusted artifacts created in the OCI repository. An empty string means the artifacts do not expire | Yes      | "1d"          |
| `trustedArtifactsDebug` | Flag to enable debug logging in trusted artifacts. Set to a non-empty string to enable | Yes      | ""            |
| `orasOptions`           | oras options to pass to Trusted Artifacts calls | Yes      | ""            |
| `sourceDataArtifact`    | Location of trusted artifacts to be used to populate data directory | Yes      | ""            |
| `taskGitUrl`            | The url to the git repo where the release-service-catalog tasks and stepactions to be used are stored | Yes      | https://github.com/konflux-ci/community-catalog.git |
| `taskGitRevision`       | The revision in the taskGitUrl repo to be used | Yes      | -             |

## Workspaces

| Name | Description |
|------|-------------|
| `data` | The workspace where the snapshot spec json file resides |

## Tasks

### update-jira-status

The main task that performs the JIRA issue updates. This task:

- Processes JIRA issues from the release data
- Moves issues from "Modified" to "On QA" status
- Adds build information as comments
- Handles trusted artifacts operations

## Prerequisites

### JIRA Authentication

The pipeline requires JIRA authentication credentials stored in a Kubernetes secret. The secret should contain:

* `token`: JIRA API token

Example secret creation:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: konflux-advisory-jira-secret
  namespace: your-namespace
type: Opaque
data:
  token: <base64-encoded-api-token>
```

### RBAC Requirements

The pipeline requires RBAC permissions to read Release resources and secrets. The service account running this pipeline needs:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: update-jira-release-reader
  namespace: your-namespace
rules:
- apiGroups: ["appstudio.redhat.com"]
  resources: ["releases"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
```

## Usage Example

This pipeline is designed to be used after builds complete. Here is an example of how to call this pipeline:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: update-jira-status-run
spec:
  pipelineRef:
    name: update-jira-status
  params:
    - name: dataPath
      value: "snapshot.json"
    - name: buildInfo
      value: "Image: quay.io/org/repo:sha256-abc123, Build ID: 12345"
    - name: taskGitRevision
      value: "main"
  workspaces:
    - name: data
      persistentVolumeClaim:
        claimName: release-data-pvc
```

## JIRA Issue Processing

The pipeline processes JIRA issues as follows:

1. **Status Check**: Only processes issues that are currently in "Modified" status
2. **Transition**: Moves issues from "Modified" to "On QA" status using JIRA transitions
3. **Comment Addition**: Adds a comment with build information and advisory link

### JIRA Transition Requirements

The pipeline requires:
* Issues to be in "Modified" status initially
* Permission to transition issues to "On QA" status
* The "On QA" transition to be available in the JIRA workflow

### Comment Format

The comment added to each issue includes:
* Build completion notification
* Build information (provided via `buildInfo` parameter)
* Status change explanation

## Error Handling

The pipeline includes comprehensive error handling:

* Validates that Release artifacts contain issue information
* Checks for JIRA authentication credentials
* Handles API failures gracefully
* Continues processing other issues if one fails
* Attempts to add comments even if status transitions fail

## Integration with Other Pipelines

This pipeline is typically used in conjunction with other pipelines in the release process:

1. **Build Pipeline**: Generates the build artifacts and build information
2. **update-jira-status Pipeline**: Updates JIRA issues with build information

## Future Enhancements

Potential improvements for this pipeline:

* Support for custom JIRA fields beyond comments
* Support for multiple JIRA instances
* Enhanced error reporting and logging
* Support for different JIRA workflows and status names
* Integration with additional build metadata sources
* Support for conditional execution based on build/advisory status 