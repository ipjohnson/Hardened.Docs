# NuGet Setup

Hardened packages are hosted on **GitHub Packages** under the `ipjohnson-org` organization. Before you can install any Hardened packages, you need to configure NuGet to authenticate with this feed.

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- A GitHub account
- A GitHub Personal Access Token (PAT) with the `read:packages` scope

### Creating a GitHub PAT

1. Go to [GitHub Settings > Developer settings > Personal access tokens > Tokens (classic)](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**
3. Give it a descriptive name (e.g., `nuget-read-packages`)
4. Select the `read:packages` scope
5. Click **Generate token**
6. Copy the token — you will not be able to see it again

!!! warning
    Keep your PAT secret. Never commit it to source control.

---

## Option 1: nuget.config (Recommended)

Add a `nuget.config` file to the root of your repository. This ensures all developers and CI pipelines use the correct package source.

```xml title="nuget.config"
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github-ipjohnson-org"
         value="https://nuget.pkg.github.com/ipjohnson-org/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github-ipjohnson-org>
      <add key="Username" value="YOUR_GITHUB_USERNAME" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github-ipjohnson-org>
  </packageSourceCredentials>
</configuration>
```

### Setting the environment variable

=== "Windows (PowerShell)"

    ```powershell
    $env:GITHUB_TOKEN = "ghp_your_token_here"
    ```

=== "Windows (Command Prompt)"

    ```batch
    set GITHUB_TOKEN=ghp_your_token_here
    ```

=== "macOS / Linux"

    ```bash
    export GITHUB_TOKEN=ghp_your_token_here
    ```

!!! tip
    Add the `GITHUB_TOKEN` export to your shell profile (`~/.bashrc`, `~/.zshrc`, etc.) so it persists across terminal sessions.

### CI/CD pipelines

In GitHub Actions, `GITHUB_TOKEN` is automatically available. Pass it as an environment variable to your build steps:

```yaml
- name: Restore
  run: dotnet restore
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

For other CI providers, store your PAT as a secret and expose it as the `GITHUB_TOKEN` environment variable.

---

## Option 2: dotnet CLI

If you prefer not to use a `nuget.config` file, you can add the source globally using the dotnet CLI:

```bash
dotnet nuget add source \
  --name github-ipjohnson-org \
  --username YOUR_GITHUB_USERNAME \
  --password ghp_your_token_here \
  --store-password-in-clear-text \
  https://nuget.pkg.github.com/ipjohnson-org/index.json
```

!!! note
    The `--store-password-in-clear-text` flag is required on macOS and Linux. On Windows, NuGet can encrypt the credential automatically without this flag.

---

## Verifying the setup

Run the following command to confirm the source is configured correctly:

```bash
dotnet nuget list source
```

You should see `github-ipjohnson-org` in the output. To verify you can fetch packages:

```bash
dotnet package search Hardened --source github-ipjohnson-org --prerelease
```

This should return a list of available Hardened packages.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `401 Unauthorized` on restore | Verify your PAT has the `read:packages` scope and has not expired |
| `Unable to load the service index` | Check the source URL is exactly `https://nuget.pkg.github.com/ipjohnson-org/index.json` |
| `%GITHUB_TOKEN%` used literally | Ensure the `GITHUB_TOKEN` environment variable is set in your current shell session |
| Packages not found | Add `--prerelease` to your search or restore — Hardened packages use pre-release versioning |

---

## Next steps

Once your NuGet source is configured, proceed to [Installation](installation.md) to add the right packages for your project.
