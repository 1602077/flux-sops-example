# flux-sops-example

bootstrap repo secret
```bash
flux bootstrap github \
  --token-auth \
  --owner=1602077 \
  --repository=flux-sops-example \
  --branch=main \
  --path=clusters/minikube \
  --personal \
  --export
```

Creates an source artifact under `gitrepository/flux-system` - check its access with `flux get source git`.

Access is authorised using the PAT token provided on running the boostrapping command. This token is stored under the flux-system namespace.
```bash
k get secret -n flux-system flux-system -o jsonpath='{.data.password}'
```


```bash
flux create secret git github-pat --password=$GITHUB_PAT \
    --username 1602077 \
    --url https://github.com/1602077/flux-sops-example \
    --namespace=flux-system
```


generate age key

```bash
age-keygen -o age.agekey
```
```
cat age.agekey |
    kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=/dev/stdin
    secret/sops-age created
```

encrypt your secret
```
sops --age=${AGE_PUBLIC_KEY} \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place basic-auth.yaml
```

create k8s kustomization for sops encrypted secret then create a flux kustomization using

flux create kustomization secrets \
--source=github-repo \
--path=./clusters/minikube/secrets \
--prune=true \
--interval=10m \
--decryption-provider=sops \
--decryption-secret=sops-age

