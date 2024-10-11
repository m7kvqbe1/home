+++
title = 'Move Issue to Project Column'
date = 2024-10-10T12:08:10+01:00
draft = false
+++

[**m7kvqbe1/github-action-move-issues**](https://github.com/marketplace/actions/move-issue-to-project-column) is a GitHub Action that automates the movement of issues within GitHub Projects V2 based on specific labels.

Find the related source code here:

https://github.com/m7kvqbe1/github-action-move-issues

## Overview

- **Automatic Issue Movement**: Moves labeled issues to the specified column.
- **Supports Multiple Labels**: Triggers on multiple specified labels.
- **Column Filtering**: Configure columns to ignore or target.
- **GitHub Projects V2 Integration**: Works with the latest version of Projects.

### Example Workflow

Here's how you can configure this Action in your workflow:

```yaml
name: Move Issue on Label
on:
  issues:
    types: [labeled]

jobs:
  move-issue:
    runs-on: ubuntu-latest
    steps:
      - name: Move Issue to Project Column
        uses: m7kvqbe1/github-action-move-issues@v1.0.0
        with:
          github-token: ${{ secrets.PAT_TOKEN }}
          project-url: "https://github.com/orgs/your-org/projects/1"
          target-labels: "Size: Small, Size: Medium"
          target-column: "Candidates for Ready"
          ignored-columns: "Ready, In Progress, In Review, Done"
```

## Use Cases

- **Manage Backlogs**: Automatically move labeled issues to a "Ready for Review" column.
- **Reduce Manual Work**: Automate moving issues between workflow stages.
- **Keep Projects Updated**: Ensure issues are always in the correct column.

## Getting Started

Include this Action in your workflow to automate project board updates. Create a PAT (Personal Access Token) with the `repo` and `admin:org` scopes and add it to your repository secrets for secure access.

## How It Works

This GitHub Action leverages the GitHub GraphQL API to interact with Projects V2 and perform the following steps:

1. **Validate Issue**: Ensures the issue that triggered the workflow has the required label.

```go
const validateIssue = (issue, TARGET_LABELS, core) => {
  if (!issue || !issue.node_id) {
    core.setFailed("Invalid or missing issue object");
    return false;
  }

  if (!issue.labels.some((label) => TARGET_LABELS.includes(label.name))) {
    console.log(`Issue #${issue.number} does not have a target label`);
    return false;
  }

  return true;
};
```

2. **Fetch Project Data**: Uses the GitHub GraphQL API to find the project specified by the `project-url` input. This involves fetching all projects in the organization and finding the one that matches the given URL.

```go
const fetchAllProjects = async (
  octokit,
  orgName,
  cursor = null,
  allProjects = []
) => {
  const query = `
    query($orgName: String!, $cursor: String) {
      organization(login: $orgName) {
        projectsV2(first: 100, after: $cursor) {
          nodes { id, url, number }
          pageInfo { hasNextPage, endCursor }
        }
      }
    }
  `;

  const result = await octokit.graphql(query, { orgName, cursor });

  const updatedProjects = [
    ...allProjects,
    ...result.organization.projectsV2.nodes,
  ];

  if (result.organization.projectsV2.pageInfo.hasNextPage) {
    return fetchAllProjects(
      octokit,
      orgName,
      result.organization.projectsV2.pageInfo.endCursor,
      updatedProjects
    );
  }

  return updatedProjects;
};
```

3. **Check Current Status**: Once the project is identified, the action checks the current status of the issue to determine if it needs to be moved. If the issue is in a column that should be ignored, it skips the update.

```go
const getCurrentStatus = (issueItemData) => {
  return issueItemData.fieldValues?.nodes.find(
    (node) => node.field?.name === "Status"
  )?.name;
};

const processIssueItem = async (
  octokit,
  projectData,
  issue,
  TARGET_COLUMN,
  IGNORED_COLUMNS,
  core
) => {
  // ...

  const currentStatus = getCurrentStatus(issueItemData);

  if (IGNORED_COLUMNS.includes(currentStatus)) {
    throw new Error(
      `Issue #${issue.number} is in an ignored column (${currentStatus}). Skipping.`
    )
  }

  await updateIssueStatus(
    octokit,
    projectData.id,
    issueItemData.id,
    statusField.id,
    targetStatusOption.id
  );
  console.log(`Moved issue #${issue.number} to "${TARGET_COLUMN}"`);
};
```

4. **Update Issue Status**: Moves the issue to the desired column if needed, using the GitHub GraphQL mutation API.

```go
const updateIssueStatus = async (
  octokit,
  projectId,
  itemId,
  statusFieldId,
  statusOptionId
) => {
  const mutation = `
    mutation($projectId: ID!, $itemId: ID!, $statusFieldId: ID!, $statusOptionId: String!) {
      updateProjectV2ItemFieldValue(
        input: {
          projectId: $projectId
          itemId: $itemId
          fieldId: $statusFieldId
          value: { singleSelectOptionId: $statusOptionId }
        }
      ) {
        projectV2Item { id }
      }
    }
  `;

  await octokit.graphql(mutation, {
    projectId,
    itemId,
    statusFieldId,
    statusOptionId,
  });
};
```

## Building and Publishing the Action

To make this GitHub Action available for others:

### Building with @vercel/ncc

1. Install @vercel/ncc:

```bash
npm install --save-dev @vercel/ncc
```

2. Add to `package.json`:

```json
{
  "scripts": {
    "build": "ncc build index.js -o dist"
  }
}
```

3. Run: `npm run build`

This creates a `dist/index.js` file with your action and dependencies.

### Publishing to the GitHub Actions Marketplace

1. Update `action.yml`:

```yaml
runs:
  using: "node20"
  main: "dist/index.js"
```

2. Commit changes, including `dist`.
3. Create a new release with a semantic version tag.
4. Publish to Marketplace:
   - Edit the release on GitHub
   - Check "Publish this Action to the GitHub Marketplace"
   - Fill required information and publish

Remember to update the action version in workflows:

```yaml
- uses: m7kvqbe1/github-action-move-issues@v1.0.1
```
