---
layout: post
title: "Using Sops (Secrets OPerationS) with Helm: Plugin-less solution"
author: "Zakaria"
comments: true
description: Helm with sops without plugings
---
# Background:

If you are a [Helm](https://github.com/helm/helm) (Kubernetes package manager) user, you know that Helm provides no default way of protecting secrets, especially if all the senstive values are stored along with the Helm Chart in the same repository. [helm-secrets](https://github.com/zendesk/helm-secrets) plugin can help with that, but one can also roll up his custom solution. `helm-secrets` forces the user to use a specific directory structure (`helm_vars` directory that holds all env and secret variables). This may not seem like a bummer to some, but it can be an enough reason for others to ditch the plugin. 

`helm-secrets` can be seen as a Helm wrapper for [Sops](https://github.com/mozilla/sops), a tool that allows protecting your sensitive files through cryptography. Using [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard), Sops encrypts files (several formats are supported like Yaml and Json), and also appends metadata that will help in the decryption later. The use cases for Sops can vary, but one of the most common usage is securing files stored in versioning systems like git. 

# Encrypting / Decrypting secrets

Lets's suppose we have the following secrets file that we would like to use in our Helm Chart

__secret_example.yaml__
```
SECRET_VALUE1: secret_value1
SECRET_VALUE2: secret_value2
```

The result of encrypting the file using a [gpg2](https://gnupg.org/) key (running: `sops -e -i --pgp 30FE2103DB8D388B0BDBFC2222C46635D8ED0E9 secret-example.yaml`):

```
SECRET_VALUE1: ENC[AES256_GCM,data:UzqJmLcBZWSBS119ZQ==,iv:x6P1RIMnNAXc0FXQthAAUG23rdM5pZPizAYYh5eqfQg=,tag:Ack/ltD57S69QBlHt4ehqw==,type:str]
SECRET_VALUE2: ENC[AES256_GCM,data:Ah+U8rMw7jqfHwL32g==,iv:AMbSUtzj1XFjR1agKGpZyms3Dxhqm+wdnSWc5FWiqMI=,tag:LWdhBypGitnRU/xLtM0Q+w==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    lastmodified: '2020-11-21T20:00:27Z'
    mac: ENC[AES256_GCM,data:Ii2i5V9uX+hxF7bE/esYwGqoUwojtm+vrzWSIFdQe3dZ0d353qp13PRKOcsiQUTJVzXwEszwA6SWJTl/lvA0vmkN+sX0HISsfBvcgBlE1QPxLoG46ST71/rJhUY3z1RfDlOQFx4QMYMRSStzFSSOrkqR1krFpEZAA0tK2jV0h9E=,iv:jduQPej9/DPkDnwGBJc7Wrnf/XB/NiqkPUiuMcYyrn8=,tag:LP41QIlD6H0F05WFgHgrYg==,type:str]
    pgp:
    -   created_at: '2020-11-21T20:00:27Z'
        enc: |-
            -----BEGIN PGP MESSAGE-----

            wcDMA7p1B6ApVZDfAQwAVyuMdmxnKMwyC/lf7QyKM0HxWBRszhZOb5njCDtasemy
            4646VfwYRBECGMKn0kU0zwZqmHV8o9DJd4NPHTAtB6HKuMurt8mt1YUVQU2+kaL2
            K7EWrWbuVwKR55Bv55ORGI5HNIy49jtSS/qdfIjgTO18dBgap/FOkK5fBNgxbNXI
            nqstcsqFT5Qsqb60q5rR7vaC0Riy6nqQM+slHW2ippSc1LgfCmxvNuDJw89oNLbq
            nrEBmkuYoyxAROuIDLkNiNujFvZ8ei/5vDdw7q41v9vkt4oEt2hjO9sy7EUho6ry
            m8PH1u3/w6Cz8Tg2We0rh6w94TTw/8Ukkxw7lBoZ2VCoc/sf+1Z1YDfC0kTlBqj9
            SUxBCp0jViDKShijDlFvMhjcFNJZuCe2yecR6d2qh700o9SckTdKWPMEvWUp07ND
            ZCy8Ko4yOc6UrN6kqKN7rHk7nLjVrlf7t3aJ9VGalFDXO/CZyqKWvN6oTXOQB4Pw
            hafSv63nWFGBTYl5lOin0uYBPgZEn8IcaWO1+jF4Kg2aCp1xln5Pcc4+lzhxfkiJ
            OQ2qF1WNCMAici5S2htNt5XEAP+I6iB6l1ukijqNqspq5AhcUOWWTPyAwzO9fOD4
            J/ri1X4sHwA=
            =7E7x
            -----END PGP MESSAGE-----
        fp: 30FE2103DB8D388B0BDBFC2222C46635D8ED0E9
    unencrypted_suffix: _unencrypted
    version: 3.6.1

```

__Explanation__: Sops encrypted the values in the original file and added a `sops` metadata section at the bottom with references to the type and the id of the key used to encrypt the values

Now the secret can be safely stored (in git) along with the Helm Chart. 

The secret file values can be decrypted before deploying/updating the Helm Chart, and then used to create the Kubernetes secrets.  

# Secret Helm Template

Let's suppose we are storing all of our secret files in a folder named `secrets` in the root path of the Helm Chart, then using a simple bash script, we can loop through the files and decrypt them. 


```
ls -1 secrets/ | while read file; do sops -d --pgp $GPG_KEY_ID secrets/$file > secrets/${file:0:(-5)}.dec.yaml; done

```

__Explanation__: We have given the files the `.dec.yaml` extension so that we can easily filter them from our Helm template, e.g `secret-example.yaml` will be decrypted as `secret-example.dec.yaml`


We are now ready to generate our Kubernetes secrets using the Helm template. 

```
{% raw %}
{{- $files := .Files }}
{{ range $path, $bytes := .Files.Glob "secrets/**.dec.yaml" }}
{{- $secretData := $files.Get $path | fromYaml }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ trimPrefix "." (base $path | replace ".dec.yaml" "") }}-secret
  namespace: default
type: Opaque
data:
{{- range $name, $secret := ($secretData) }}
  {{ $name }}: {{ $secret | b64enc }}
{{- end }}
---
{{- end }}
{% endraw %}
```

__Explanation__: We loop through all the files with `.dec.yaml` extention in the `secret` directory and we create a secret with the name filename-without-extention-secret, e.g `secret-example-secret`

For convenience, we can put our script into a file e.g `decrypt-secrets.sh` and call it whenever we want to call `helm install` or `helm upgrade`

it's also recommended to remove the `.dec.yaml` files after the release is done.

__Take away__:  if one knows how both Helm and Sops work, he can simply do without the plugin that can add unecessary complexity.