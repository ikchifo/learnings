# Managing secrets with 1Password CLI + op:// references

A reference for replacing hardcoded secrets in dotfiles and tool
configs with 1Password CLI lookups, keeping secrets out of plaintext
files on disk.

---

## The problem

Hardcoding API tokens, credentials, and secrets directly in
`.zshrc`, `.env`, or tool config files means:

- Secrets live in plaintext on disk indefinitely
- They end up in shell history, dotfile backups, and git repos
- AI agents (Claude Code, Copilot) can read them from env or files
- Rotation requires finding and updating every hardcoded occurrence

## The approach

Use the 1Password CLI (`op`) to resolve secrets at runtime from
your 1Password vault. Secrets are stored once in 1Password and
referenced via `op://` URIs.

### op:// URI format

```
op://<vault>/<item>/<field>
```

Examples:

```
op://Private/my-api-key/credential
op://Company/ArgoCD Kubeflow/API Key
op://Work/database/password
```

### Prerequisites

1. Install 1Password CLI: `brew install 1password-cli`
2. Enable desktop app integration:
   **1Password > Settings > Developer > Integrate with 1Password
   CLI**
3. Verify: `op vault list` should list your vaults

If you have multiple accounts, use `--account` to disambiguate:

```bash
op vault list --account my.1password.com
```

## Patterns

### Shell env vars (.zshrc)

Replace hardcoded exports with `op read`:

```bash
# Before (secret in plaintext on disk)
export ARGOCD_API_TOKEN="eyJhbGciOi..."

# After (resolved from 1Password at shell startup)
export ARGOCD_API_TOKEN="$(op read \
  'op://Company/ArgoCD Kubeflow/API Key' \
  --account my.1password.com 2>/dev/null)"
```

The `2>/dev/null` suppresses errors when 1Password is locked or
the desktop app is not running — the variable will be empty rather
than spewing errors on every shell open.

### Claude Code MCP server config

MCP server configs in `~/.claude.json` support `${ENV_VAR}`
expansion. Combine with the shell pattern above:

```json
{
  "env": {
    "ARGOCD_API_TOKEN": "${ARGOCD_API_TOKEN}"
  }
}
```

The env var is resolved by the shell (via `op read`), then Claude
Code expands `${ARGOCD_API_TOKEN}` when starting the MCP server.
The token never appears in the Claude config file itself.

### Direct op:// injection (tools that support it)

Some tools can resolve `op://` URIs natively via `op run`:

```bash
op run --account my.1password.com -- my-tool --token op://Vault/Item/Field
```

Or via environment file:

```bash
# .env (safe to commit — contains only references, not values)
ARGOCD_API_TOKEN=op://Company/ArgoCD Kubeflow/API Key

# Resolve and run
op run --env-file .env -- my-command
```

### Varlock integration

Varlock can use 1Password as a secret backend via the
`@varlock/1password-plugin`:

```bash
# .env.schema
# @plugin(@varlock/1password-plugin)
# @sensitive @required
ARGOCD_API_TOKEN=op://Company/ArgoCD Kubeflow/API Key
```

Then `varlock run -- command` or `varlock load --format shell`
resolves secrets at runtime.

## Common operations

### Store a new secret

```bash
op item create \
  --account my.1password.com \
  --vault Private \
  --category "API Credential" \
  --title "My Service Token" \
  "credential=the-secret-value"
```

### Update an existing secret

```bash
op item edit "Item Name" \
  --account my.1password.com \
  --vault Company \
  "API Key=new-token-value"
```

### Read a secret (for scripting)

```bash
op read 'op://Vault/Item/Field' --account my.1password.com
```

### Find an item

```bash
# List all items in a vault
op item list --account my.1password.com --vault Private

# Search by title
op item list --account my.1password.com | grep -i argocd
```

### List fields in an item

```bash
op item get "Item Name" --account my.1password.com --vault Company
```

## Gotchas

- **Shell startup latency**: `op read` adds ~200-500ms per call to
  shell startup. If you have many secrets, batch them or use `op
  run` with an env file instead.
- **Biometric/unlock required**: `op read` will prompt for
  biometric auth if 1Password is locked. This is by design but can
  be surprising in non-interactive contexts (cron, CI).
- **Multiple accounts**: Always pass `--account` when you have more
  than one 1Password account to avoid ambiguity errors.
- **Empty on failure**: With `2>/dev/null`, a failed `op read`
  silently sets the variable to empty. Tools that require the token
  will fail later with an auth error rather than at shell startup.

## References

- [1Password CLI docs](https://developer.1password.com/docs/cli/)
- [1Password secret references](https://developer.1password.com/docs/cli/secret-references/)
- [op run](https://developer.1password.com/docs/cli/reference/management-commands/run/)
- [Varlock](https://varlock.dev/)
