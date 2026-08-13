---
title: "Installing Let’s Encrypt certificates in OpenShift"
date: 2021-10-04T16:11:53+02:00
draft: false
showtoc: false
tags: ["OpenShift"]

---

# Introduction
At my homelab I deploy and destroy OpenShift clusters several times a day. One of the most annoying things for me is accessing OpenShift console when it doesn't have proper certificates installed: browser warns about self-signed certificates and makes me do several clicks first in order to access the UI. And of course, in real-life deployments you have to use proper certificates to secure routes and API endpoints.

These days it's easy to obtain free TLS certificates using Let's Encrypt or similar services. In March 2018 Let's Encrypt [added](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579) support for wildcard certificates, that made it finally possible to use it for my OpenShift deployments. I use `acme.sh` together with [DNS API](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) for [DNS challenge](https://letsencrypt.org/docs/challenge-types/).

# Solution
Usually I have 2 different versions of clusters running at the same time: stable and development. So I need to request a wildcard certificate for my main domain plus for api and *.apps subdomains for both clusters:

```shell
$ export DOMAIN=my_domain.fqdn
$ acme.sh --issue --dns dns_dreamhost \
        -d "$DOMAIN" \
        -d "*.$DOMAIN" \
        -d "api.stable.$DOMAIN" \
        -d "api.devel.$DOMAIN" \
        -d "*.apps.stable.$DOMAIN" \
        -d "*.apps.devel.$DOMAIN"
```

## Installing the certificate
Once you successfully receive the certificate, it's time to install it. There are 2 places where we need to introduce new certificates: ingress controller and API server.

### Ingress controller
Create a secret that contains full certificate chain and private key in the `openshift-ingress` namespace:
```shell
$ oc create secret tls letsencrypt-certs -n openshift-ingress \
    --cert=${HOME}/.acme.sh/${DOMAIN}/fullchain.cer \
    --key=${HOME}/.acme.sh/${DOMAIN}/${DOMAIN}.key \
    --dry-run=client -o yaml | oc apply -f -

```

Then update the ingress controller to use the created secret:

```shell
$ oc patch ingresscontroller default -n openshift-ingress-operator \
    --type=merge --patch='{"spec": { "defaultCertificate": { "name": "letsencrypt-certs" }}}'
```

### API server
Same for the API server:
Create a secret that contains full certificate chain and private key in the `openshift-config` namespace:

```shell
$ oc create secret tls letsencrypt-certs -n openshift-config \
    --cert=${HOME}/.acme.sh/${DOMAIN}/fullchain.cer \
    --key=${HOME}/.acme.sh/${DOMAIN}/${DOMAIN}.key \
    --dry-run=client -o yaml | oc apply -f -
```

And update the API server with the new secret reference. Repeat this for each cluster (`stable`, `devel`):
```shell
$ for CLUSTER in stable devel; do
    oc patch apiserver cluster --type=merge \
      -p '{"spec":{"servingCerts": {"namedCertificates":  [{"names": ["api.'${CLUSTER}'.'${DOMAIN}'],  "servingCertificate": {"name": "letsencrypt-certs"}}]}}}'
  done
```

> **Note:** Patching the API server triggers a rolling restart of the `kube-apiserver` pods. During this rollout (typically 5-15 minutes), `oc` commands may fail with connection errors. This is expected — wait for the rollout to complete before proceeding.

## Verifying the installation
Once the rollout is complete, verify that the new certificates are in place:

```shell
$ echo | openssl s_client -connect "api.${CLUSTER}.${DOMAIN}:6443" -servername "api.${CLUSTER}.${DOMAIN}" 2>/dev/null | openssl x509 -noout -issuer -dates
issuer=C = US, O = Let's Encrypt, CN = R3
notBefore=...
notAfter=...
```

Check the console route certificate:
```shell
$ echo | openssl s_client -connect "console-openshift-console.apps.${CLUSTER}.${DOMAIN}:443" -servername "console-openshift-console.apps.${CLUSTER}.${DOMAIN}" 2>/dev/null | openssl x509 -noout -issuer -dates
```

Both should show Let's Encrypt as the issuer. After that you should be able to connect to the cluster console without any certificate warnings.

## Fixing `oc` client
You might encounter one issue with the CLI tools if they use an old KUBECONFIG. If it references old CA data, you won't be able to connect to the cluster:
```shell
$ oc whoami
Unable to connect to the server: x509: certificate signed by unknown authority

```

The solution is to remove the old CA data from your KUBECONFIG:
```
$ sed -i -e "s/\(certificate-authority-data:\).*//" $KUBECONFIG
```

> **Warning:** This removes CA data for **all** clusters in that KUBECONFIG. It works because Let's Encrypt certificates are signed by a publicly trusted CA (already in your system trust store). If your KUBECONFIG contains entries for clusters that still use self-signed certificates, those will break. In that case, target only the specific cluster context instead.

And after that connection succeeds:
```
$ oc whoami
system:admin
```

## Certificate renewal
Let's Encrypt certificates expire after 90 days. `acme.sh` installs a cron job automatically to renew certificates, but you still need to re-apply them to OpenShift after each renewal.

Create a renewal script (e.g. `~/bin/renew-ocp-certs.sh`):

```shell
#!/bin/bash
set -euo pipefail

DOMAIN=my_domain.fqdn
CERT_DIR="${HOME}/.acme.sh/${DOMAIN}"

for NS in openshift-ingress openshift-config; do
  oc create secret tls letsencrypt-certs -n "${NS}" \
    --cert="${CERT_DIR}/fullchain.cer" \
    --key="${CERT_DIR}/${DOMAIN}.key" \
    --dry-run=client -o yaml | oc apply -f -
done

echo "Secrets updated. API server and ingress controller will pick up the new certs."
```

Then configure `acme.sh` to run it after successful renewal:

```shell
$ acme.sh --install-cert -d "$DOMAIN" \
    --reloadcmd "~/bin/renew-ocp-certs.sh"
```

`acme.sh` will call `--reloadcmd` automatically after each successful renewal via its cron job. No manual intervention needed.

### References:
* https://docs.openshift.com/container-platform/latest/security/certificates/replacing-default-ingress-certificate.html
* https://docs.openshift.com/container-platform/latest/security/certificates/api-server.html
* https://github.com/acmesh-official/acme.sh/wiki/Using-pre-hook-post-hook-reloadcmd
