# Global rules (Kilo Code)

- Never read, print, or search .env files, kubeconfig, or talosconfig.
- Before any kubectl apply/delete or talosctl apply, show the exact command
  and explain what it will change. Prefer --dry-run=server first.
- Read-only inspection (get, describe, logs) is fine to run freely.
- Prefer declarative changes in the repo over imperative cluster edits.
- Explain destructive or hard-to-reverse changes before proposing them.
