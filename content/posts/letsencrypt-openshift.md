---
title: "Installing Let’s Encrypt certificates in OpenShift"
date: 2021-10-04T16:11:53+02:00
draft: false
showtoc: false
tags: ["OpenShift"]

---

# Introduction
At my homelab I deploy and destroy OpenShift clusters several times a day. One of the most annoying things for me is accessing OpenShift console when it doesn't have proper certificates installed: browser warns about self-signed certificates and makes me do several clicks first in order to access the UI. And of course, in real-life deployments you have to use proper certificates to secure routes and API endpoints.

These days it's easy to obtain free TLS certificates using Let's Encrypt or similar services. In March 2018 Let's Encrypt [added](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579) support for wildcard certificates, that made it finally possible to use it for my OpenShift deployments. I use `acme.sh` together with [DNS API](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) for [DNS chanllenge](https://letsencrypt.org/docs/challenge-types/).

# Solution
Usually I have 2 different versions of clusters running at the same time: stable and development. So I need to request a wildcard certificate for my main domain plus for api and *.apps subdomains for both clusters:

```shell
$ export DOMAIN=my_domain.fqdn
$ acme.sh --issue --dns dns_dreamhost \
        -d '$DOMAIN' \
        -d '*.$DOMAIN' \
        -d 'api.stable.$DOMAIN' \
        -d 'api.devel.$DOMAIN' \
        -d '*.apps.stable.$DOMAIN' \
        -d '*.apps.devel.$DOMAIN'
```

## Installing the certificate
Once you sucessfully receive the certificate, it's time to install it. There are 2 places where we need to introduce new certificates: ingress controller and API server.

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

And update the API server with the new secret reference:
```shell
$ export CLUSTER=stable
$ oc patch apiserver cluster --type=merge \
    -p '{"spec":{"servingCerts": {"namedCertificates":  [{"names": ["api.'${CLUSTER}'.'${DOMAIN}"],  "servingCertificate": {"name": "letsencrypt-certs"}}]}}}' 
```


It will take some time to rebuild the containers and deploy the pods, but once it's done you would be able to connect to the cluster console without any warnings.

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

And after that connection succeeds:
```
$ oc whoami
system:admin
```


### References:
* https://docs.openshift.com/container-platform/4.8/security/certificates/replacing-default-ingress-certificate.html
* https://docs.openshift.com/container-platform/4.8/security/certificates/api-server.html
