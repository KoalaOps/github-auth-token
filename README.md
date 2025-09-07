# GitHub Auth Token

Generates GitHub tokens for cross-repository access using GitHub Apps or Personal Access Tokens.

## Features

- 🔑 **Dual authentication** - GitHub App or PAT
- 🔄 **Cross-repo access** - Access private repositories
- 🎯 **Smart fallback** - Uses GITHUB_TOKEN if no auth provided
- 🔒 **Secure** - No credentials in logs

## Usage

```yaml
- name: Get GitHub token
  uses: KoalaOps/github-auth-token@v1
  id: auth
  with:
    github_app_id: ${{ secrets.GH_APP_ID }}
    github_app_private_key: ${{ secrets.GH_APP_PK }}

- name: Checkout private repo
  uses: actions/checkout@v4
  with:
    repository: myorg/private-repo
    token: ${{ steps.auth.outputs.token }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|----------|
| `github_app_id` | GitHub App ID | ❌* | - |
| `github_app_private_key` | GitHub App private key (PEM format) | ❌* | - |
| `github_pat` | Personal Access Token | ❌* | - |
| `repository` | Repository to authenticate against (GitHub App only) | ❌ | Current repo |
| `fallback_token` | Token to use as fallback | ❌ | `${{ github.token }}` |

*Provide either GitHub App credentials OR PAT. If neither provided, uses fallback_token.

**Note:** The `repository` input is only used when authenticating with a GitHub App to scope the token to specific repositories. It has no effect when using PAT or fallback authentication.

## Outputs

| Output | Description |
|--------|-------------|
| `token` | GitHub authentication token |
| `token_type` | Type of token (app, pat, or github_token) |

## Authentication Priority

1. **GitHub App** (if app_id and private_key provided)
2. **Personal Access Token** (if pat provided)
3. **Fallback token** (defaults to GITHUB_TOKEN)

## Examples

### GitHub App authentication
```yaml
- name: Setup GitHub App auth
  uses: KoalaOps/github-auth-token@v1
  id: auth
  with:
    github_app_id: ${{ secrets.GH_APP_ID }}
    github_app_private_key: ${{ secrets.GH_APP_PK }}

- name: Clone deployment repo
  uses: actions/checkout@v4
  with:
    repository: myorg/k8s-deployments
    token: ${{ steps.auth.outputs.token }}
    path: deployments
```

### PAT authentication
```yaml
- name: Setup PAT auth
  uses: KoalaOps/github-auth-token@v1
  id: auth
  with:
    github_pat: ${{ secrets.GHA_PAT }}

- name: Create PR in another repo
  env:
    GH_TOKEN: ${{ steps.auth.outputs.token }}
  run: |
    gh pr create --repo myorg/other-repo \
      --title "Update from workflow" \
      --body "Automated update"
```

### Conditional cross-repo access
```yaml
- name: Get auth token
  uses: KoalaOps/github-auth-token@v1
  id: auth
  with:
    # Only use PAT for cross-repo, otherwise default token
    github_pat: ${{ inputs.deployment_repo != github.repository && secrets.GHA_PAT || '' }}

- name: Checkout
  uses: actions/checkout@v4
  with:
    repository: ${{ inputs.deployment_repo || github.repository }}
    token: ${{ steps.auth.outputs.token }}
```

## Setup

### Creating a GitHub App

1. Go to Settings → Developer settings → GitHub Apps
2. Create new GitHub App with permissions:
   - Contents: Read & Write
   - Pull requests: Write (if needed)
   - Actions: Read (if needed)
3. Generate a private key
4. Install the app on your organization/repositories
5. Store as secrets:
   - `GH_APP_ID`: The App ID
   - `GH_APP_PK`: The private key (full PEM content)

### Creating a PAT

1. Go to Settings → Developer settings → Personal access tokens
2. Generate new token (classic) with scopes:
   - `repo` (full control of private repositories)
   - `workflow` (if updating workflows)
3. Store as secret: `GHA_PAT`

## Security Notes

- GitHub App tokens expire after 1 hour
- PATs should have minimal required scopes
- Never log tokens in workflow output
- Use GitHub App over PAT when possible (better security)
- Tokens are masked in logs automatically

## Troubleshooting

### "Bad credentials" error
- Verify the GitHub App is installed on target repository
- Check PAT hasn't expired
- Ensure correct secret names

### "Resource not accessible by integration"
- Default GITHUB_TOKEN lacks cross-repo permissions
- Use GitHub App or PAT for cross-repo access

### Token expiration
- GitHub App tokens are short-lived (1 hour)
- For long workflows, regenerate token if needed