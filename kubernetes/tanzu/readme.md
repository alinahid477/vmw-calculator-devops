# Apply private regustry trust at tanzu namespace level

## get the self signed cert of the pvt registry

run command to get the cert
```
penssl s_client -connect IPorFQDN:443
```

## Convert to base 64

From the output of the above command grab the content starting from ---begin certificate (including the '---begin certificate') ending ---end certificate (including '---end-certificate---')

head to https://base64.guru/converter/encode/text and convert to base64.

Then grab the base64 encoded value and apply to trust-private-registry.yaml


## Apply

- Login into supervisor cluster
    ```
    kubectl-vsphere login --insecure-skip-tls-verify --server sddc.private.local -u administrator@vsphere.local
    ```
- Switch context to target namespace
    ```
    kubectl config use-context mytanzunamespace
    ```
- `kubectl apply -f trust-private-registry.yaml`
