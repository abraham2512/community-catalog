# update-jira-on-qa (Task)

A Tekton `Task` that moves JIRA issues from "Modified" to "On QA" status and adds build information as comments after a successful build and advisory publication.

## Overview

This task is designed to automatically update JIRA issues when builds complete successfully and advisories are published. It:

1. Extracts JIRA issue information from the Release artifacts
2. Checks if issues are in "Modified" status
3. Moves issues from "Modified" to "On QA" status
4. Adds a comment with build information and advisory link

The task supports authentication to JIRA using credentials stored in Kubernetes secrets and can handle multiple JIRA issues per release.

## Parameters

| Name            | Description                                                           | Optional | Default value |
|-----------------|-----------------------------------------------------------------------|----------|---------------|
| `dataPath`      | Path to the JSON string of the merged data to use in the data workspace | No       | -             |
| `advisoryUrl`   | The URL of the advisory the issues were fixed in. This is added in a comment on the issue | No       | -             |
| `buildInfo`     | Information about the build that was generated (e.g., image digest, build ID, etc.) | No       | -             |
| `jiraSecretName`| Name of secret which contains JIRA authentication credentials         | No       | -             |
| `jiraUrl`       | The base URL of the JIRA instance (e.g., https://issues.redhat.com)   | No       | -             |

## Results

| Name               | Description                                    |
|-------------------|------------------------------------------------|
| `sourceDataArtifact`| Produced trusted data artifact                 |

## Dependencies

The task runs on a `quay.io/konflux-ci/release-service-utils` base image and requires:

* **`kubectl`**: Used to read Release information from the Kubernetes cluster
* **`jq`**: Used to parse JSON data from Release artifacts and JIRA responses
* **`curl`**: Used to make HTTP requests to JIRA API

## JIRA Authentication

The task requires JIRA authentication credentials stored in a Kubernetes secret. The secret should contain:

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

## Usage Example

This task is designed to be used as a step within a larger Tekton Pipeline. Here is an example of how to call this task:

```yaml
- name: update-jira-on-qa
  runAfter: ["build-and-push", "publish-advisory"]
  taskRef:
    name: update-jira-on-qa
  params:
    - name: dataPath
      value: "snapshot.json"
    - name: advisoryUrl
      value: "https://access.redhat.com/errata/RHBA-2024-1234"
    - name: buildInfo
      value: "Image: quay.io/org/repo:sha256-abc123, Build ID: 12345"
  workspaces:
    - name: data
      workspace: release-data
```

## JIRA Issue Processing

The task processes JIRA issues as follows:

1. **Status Check**: Only processes issues that are currently in "Modified" status
2. **Transition**: Moves issues from "Modified" to "On QA" status using JIRA transitions
3. **Comment Addition**: Adds a comment with build information and advisory link

### JIRA Transition Requirements

The task requires:
* Issues to be in "Modified" status initially
* Permission to transition issues to "On QA" status
* The "On QA" transition to be available in the JIRA workflow

### Comment Format

The comment added to each issue includes:
* Build completion notification
* Build information (provided via `buildInfo` parameter)
* Advisory URL link
* Status change explanation

## Error Handling

The task includes comprehensive error handling:

* Validates that Release artifacts contain issue information
* Checks for JIRA authentication credentials
* Handles API failures gracefully
* Continues processing other issues if one fails
* Attempts to add comments even if status transitions fail

## RBAC Requirements

The task requires RBAC permissions to read Release resources and secrets. The service account running this task needs:

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

## Future Enhancements

Potential improvements for this task:

* Support for custom JIRA fields beyond comments
* Support for multiple JIRA instances
* Enhanced error reporting and logging
* Support for different JIRA workflows and status names
* Integration with additional build metadata sources 