# flux-sops-example

bootstrap repo secret

```bash
flux create secret git github-pat --password=$GITHUB_PAT \
    --username 1602077 \
    --url https://github.com/1602077/flux-sops-example \
    --namespace=flux-system
```

