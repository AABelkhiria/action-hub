# Unified Versioning Action

This GitHub Action calculates the next version for your project based on different versioning strategies. It supports Calendar Versioning (CalVer) and Semantic Versioning (SemVer) for both Git tags and GitHub Container Registry (GHCR) image tags.

## Usage

To use this action, add the following step to your GitHub Actions workflow file (e.g., `.github/workflows/release.yml`):

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0 # Required to fetch all history for git-based versioning

      - name: Calculate Next Version
        id: versioning
        uses: AABelkhiria/next-version@v1
        with:
          mode: 'git-semver' # Or 'git-calver', 'ghcr-calver'
          # See inputs below for more options

      - name: Create Git Tag
        run: |
          echo "Creating new tag: ${{ steps.versioning.outputs.new-version }}"
          git tag ${{ steps.versioning.outputs.new-version }}
          git push origin ${{ steps.versioning.outputs.new-version }}
```

## Inputs

The action accepts the following input parameters:

| Parameter             | Description                                                                                             | Required | Default         |
| --------------------- | ------------------------------------------------------------------------------------------------------- | -------- | --------------- |
| `mode`                | The versioning mode to use. Options: `git-calver`, `ghcr-calver`, `git-semver`.                         | **Yes**  | `N/A`           |
| `ghcr-package-name`   | The full name of the package in GHCR (e.g., `your-org/your-app`). Required for `ghcr-calver` mode.      | No       | `N/A`           |
| `calver-reset-policy` | For CalVer modes, defines if the build number resets. Options: `monthly`, `continuous`.                 | No       | `monthly`       |
| `semver-increment`    | Which part of the version to increment for `git-semver` mode. Options: `major`, `minor`, `patch`.       | No       | `patch`         |
| `use-pr-labels`       | If `true`, determines `semver-increment` from PR labels (`release:major`, `release:minor`).             | No       | `false`         |
| `github-token`        | GitHub token for authenticating to GHCR. Required for `ghcr-calver` mode.                               | No       | `${{ github.token }}` |
| `initial-version`     | The version to use if no previous tags are found.                                                       | No       | `0.0.0`         |
| `override-version`    | A specific version to use as the base, overriding auto-detection from Git or GHCR.                      | No       | `N/A`           |

## Outputs

The action produces the following outputs:

| Output               | Description                                                    |
| -------------------- | -------------------------------------------------------------- |
| `new-version`        | The calculated new version string.                             |
| `previous-version`   | The latest version that was detected before incrementing.      |

## Automated Versioning with PR Labels

For `git-semver` mode, you can automate the version increment based on pull request labels. This is useful for CI/CD workflows where you want to control the version bump directly from your PRs.

To enable this, set `use-pr-labels: true`. The action will then look for the following labels on a pull request:

- `release:major`: Bumps the major version (e.g., `1.2.3` -> `2.0.0`).
- `release:minor`: Bumps the minor version (e.g., `1.2.3` -> `1.3.0`).

If neither label is found, or if the workflow is not triggered by a pull request, it will fall back to the behavior defined by the `semver-increment` input (which defaults to `patch`).

### Example Workflow for Automated Releases

This workflow runs when a pull request is merged into the `main` branch. It uses PR labels to determine the version bump, creates a new Git tag, and pushes it.

```yaml
name: Create Release on Merge
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Calculate Next Version
        id: versioning
        uses: AABelkhiria/next-version@v1
        with:
          mode: 'git-semver'
          use-pr-labels: true
          github-token: ${{ secrets.GITHUB_TOKEN }} # Pass token for API access

      - name: Create Git Tag
        run: |
          echo "Creating new tag: ${{ steps.versioning.outputs.new-version }}"
          git tag ${{ steps.versioning.outputs.new-version }}
          git push origin ${{ steps.versioning.outputs.new-version }}
```

## Versioning Strategies & Examples

### 1. `git-semver`

This mode calculates the next Semantic Version based on the latest Git tag in your repository.

**Example Workflow:**

```yaml
- name: Calculate Next SemVer Version
  id: versioning
  uses: AABelkhiria/next-version@v1
  with:
    mode: 'git-semver'
    semver-increment: 'minor' # Manually increment the minor version
```

If the latest Git tag is `v1.2.5`, this will produce:
- `new-version`: `1.3.0`
- `previous-version`: `v1.2.5`

### 2. `git-calver`

This mode calculates the next Calendar Version (`YY.MM.BUILD`) based on the latest Git tag.

- **`monthly` reset policy (default):** The build number resets to `1` each month.
- **`continuous` reset policy:** The build number increments indefinitely within the same month.

**Example Workflow:**

```yaml
- name: Calculate Next CalVer Version
  id: versioning
  uses: AABelkhiria/next-version@v1
  with:
    mode: 'git-calver'
    calver-reset-policy: 'monthly'
```

If the current date is October 2024 (`24.10`) and the latest tag is `24.10.3`, this will produce:
- `new-version`: `24.10.4`
- `previous-version`: `24.10.3`

If the latest tag was `24.9.5`, it would start fresh for the new month:
- `new-version`: `24.10.1`
- `previous-version`: `24.9.5`

### 3. `ghcr-calver`

This mode is similar to `git-calver` but calculates the next version based on the latest image tag in the GitHub Container Registry.

**Example Workflow:**

```yaml
- name: Calculate Next GHCR CalVer Version
  id: versioning
  uses: AABelkhiria/next-version@v1
  with:
    mode: 'ghcr-calver'
    ghcr-package-name: 'my-org/my-app'
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

This will check the tags for the `my-org/my-app` package in GHCR and calculate the next CalVer tag based on the same logic as `git-calver`.

## Versioning Concepts

### SemVer (Semantic Versioning)

**SemVer** is a widely adopted versioning system that gives meaning to version numbers. The format is always `MAJOR.MINOR.PATCH`.

*   **`MAJOR` (e.g., 1.2.3 -> 2.0.0):** Incremented for **incompatible** or breaking changes. When this number changes, you know you will likely have to modify your code to use the new version.
*   **`MINOR` (e.g., 1.2.3 -> 1.3.0):** Incremented for adding new **features** in a backward-compatible way. Your existing code should still work, but there's new functionality available.
*   **`PATCH` (e.g., 1.2.3 -> 1.2.4):** Incremented for making backward-compatible **bug fixes**. This is the safest type of update and doesn't add new features.

**When to use it:** It's ideal for libraries, APIs, and dependencies where developers need to know if an update will break their code.

### CalVer (Calendar Versioning)

**CalVer** is a versioning scheme that uses the project's release date as its version number. It emphasizes *when* a version was released.

The format is flexible but always based on a date. Common formats include:

*   `YYYY.MM.MICRO` (e.g., `2024.10.1`)
*   `YY.MM.DD` (e.g., `24.10.31`)
*   `YYYY.MINOR`

The final segment (like `MICRO` or `BUILD`) is a number that increases for each release within that time period (e.g., each release in the same month).

**When to use it:** It's great for user-facing applications (like web apps, operating systems, or media tools like `youtube-dl`) where the age of the release is the most important piece of information for the user. It quickly tells them how recent and up-to-date their version is.

## License

This project is licensed under the terms of the [MIT License](LICENSE).