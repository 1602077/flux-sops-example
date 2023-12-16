# flux-sops-example

Example repository for configuring in-cluster decryption of SOPS-encrypted
secrets with age using `Flux`. This closely follows Flux's [documentation](https://fluxcd.io/flux/guides/mozilla-sops/#configure-in-cluster-secrets-decryption).

## Bootstrapping Cluster

Begin by initialising the flux controller and configuring it to pull from a Git repo:
```bash
export GITHUB_OWNER=1602077
export GITHUB_REPO=flux-sops-example

flux bootstrap github \
  --token-auth \
  --owner=$GITHUB_OWNER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=clusters/minikube \
  --personal \
  --export
```

Access to your repository if private is handled via a PAT token which you will
need to provide on running the command. This token is stored in the
`flux-system` namespace under `flux-system`.

Verify your access to the repository by running `flux get source git` - there
should be a new source artefact under `gitrepository/flux-system`.

# Encrypting Secrets

`age` is recommended over `gpg` for encryption. Generate your public key by
running:
```bash
[ ! $(command -v age) ] || brew install age
AGE_PUBLIC_KEY=$(age-keygen -o age.agekey 2>&1 | awk '{print $3}')
```

**This will generate an `age.agekey` in your `pwd`. It should not be committed
to source control under any circumstance and should ideally be backed up in an
external vault.**

The public key can be committed to your repository such that other people can
encrypt keys however the `AGE-SECRET-KEY` must not as this can be used to
decrypt secrets.

To encrypt your secrets with `age` use:
```bash
sops --age=${AGE_PUBLIC_KEY} \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place basic-auth.yaml
```

`Flux` natively supports the decryption of secrets - to configure write the
signing and private key as a secret to your cluster using:
```bash
cat age.agekey |
    kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=/dev/stdin
    secret/sops-age created
```

## Decrypting Secrets

Once your secrets are encrypted with `sops` edit the `cluster` root `apps.yaml` file to configure `decryption`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/minikube
  prune: true
  wait: true
  timeout: 5m0s
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```
This references the `age` secret created earlier to handle decryption.

Or you can create an isolated `kustomization` using:
```bash
flux create kustomization secrets \
    --source=flux-system \
    --path=./apps/minikube/secrets \
    --prune=true \
    --interval=10m \
    --decryption-provider=sops \
    --decryption-secret=sops-age \
    --export
```
