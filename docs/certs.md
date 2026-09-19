# API & Ingress TLS Certificates (luke)

Two certificate surfaces are managed with ACME-issued (ZeroSSL, DNS-01 via EasyDNS)
certs. Both were in place as of the 2026-09-19 rebuild:

| Surface | Domain | Secret | Where it's referenced |
|---------|--------|--------|-----------------------|
| API server (named cert) | `api.luke.syangsao.net` | `openshift-config/api-luke-custom-cert` | `APIServer.spec.servingCerts.namedCertificates` |
| Default ingress (apps wildcard) | `*.apps.luke.syangsao.net` | `openshift-ingress/router-custom-certs` | `IngressController(default).spec.defaultCertificate` |

Both certs are **ECC** (`*_ecc` ACME dirs), issued by `ZeroSSL ECC DV SSL CA 2`.

---

## 1. Issue / renew the ACME certificates

Run from a host that has EasyDNS credentials configured in acme.sh.

```bash
# Wildcard for the apps subdomain (DNS-01)
acme.sh --issue --dns dns_easydns --domain '*.apps.luke.syangsao.net' --force

# Single-domain for the API FQDN (DNS-01)
acme.sh --issue --dns dns_easydns --domain 'api.luke.syangsao.net' --force
```

Resulting files:

```
~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer   # apps cert + CA chain
~/.acme.sh/*.apps.luke.syangsao.net_ecc/*.apps.luke.syangsao.net.key
~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer      # api cert + CA chain
~/.acme.sh/api.luke.syangsao.net_ecc/api.luke.syangsao.net.key
```

Verify the SANs before applying:

```bash
openssl x509 -in ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -ext subjectAltName
# expect: DNS:*.apps.luke.syangsao.net

openssl x509 -in ~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -ext subjectAltName
# expect: DNS:api.luke.syangsao.net
```

---

## 2. API server named certificate

The kube-apiserver serves a **named** certificate for `api.luke.syangsao.net` so
external clients can verify the API endpoint. This is a two-part change: create the
TLS secret, then point the `APIServer` resource at it.

### 2a. Create/replace the TLS secret in `openshift-config`

```bash
oc create secret tls api-luke-custom-cert \
  --cert=~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  --key=~/.acme.sh/api.luke.syangsao.net_ecc/api.luke.syangsao.net.key \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

### 2b. Set the APIServer named certificate

This is the exact `spec` currently on luke (generation 2, i.e. changed after install):

```bash
oc patch apiserver cluster --type=merge -p '{
  "spec": {
    "servingCerts": {
      "namedCertificates": [
        {
          "names": ["api.luke.syangsao.net"],
          "servingCertificate": { "name": "api-luke-custom-cert" }
        }
      ]
    }
  }
}'
```

The apiserver operator rotates the serving cert and the API becomes reachable at
`https://api.luke.syangsao.net:6443`. Verify:

```bash
oc get apiserver cluster -o jsonpath='{.spec.servingCerts.namedCertificates}'
# [{"names":["api.luke.syangsao.net"],"servingCertificate":{"name":"api-luke-custom-cert"}}]

echo | openssl s_client -connect api.luke.syangsao.net:6443 \
  -servername api.luke.syangsao.net 2>/dev/null | openssl x509 -noout -subject -dates
# subject=CN = api.luke.syangsao.net
```

> **Note:** on a fresh rebuild the `APIServer` starts with generation 1 (no named
> certs). The patch above bumps it to generation 2. This is the day-2 delta.

---

## 3. Default ingress (apps) wildcard certificate

The router serves the apps wildcard cert for all routes under
`*.apps.luke.syangsao.net` (web console, CLI, MTV UI, etc.).

### 3a. Create/replace the TLS secret in `openshift-ingress`

```bash
oc create secret tls router-custom-certs \
  --cert=~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  --key=~/.acme.sh/*.apps.luke.syangsao.net_ecc/*.apps.luke.syangsao.net.key \
  -n openshift-ingress --dry-run=client -o yaml | oc apply -f -
```

### 3b. Point the IngressController at it

The `default` IngressController (in `openshift-ingress-operator`) references this
secret as its `defaultCertificate`:

```bash
oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge -p '{ "spec": { "defaultCertificate": { "name": "router-custom-certs" } } }'
```

Verify:

```bash
oc get secret router-custom-certs -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -dates -ext subjectAltName
# subject=CN = *.apps.luke.syangsao.net

oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate}'
# {"name":"router-custom-certs"}
```

The router deployment picks up the new cert and re-rolls; existing routes keep
their hostnames but now terminate with the wildcard cert.

---

## 4. Automated path (preferred)

Both surfaces are also driven by the Ansible playbooks in
[`syangsao/openshift-certs`](https://github.com/syangsao/openshift-certs):

- `update_apps_cert.yml` — updates the **apps** ingress cert from `~/.acme.sh/`.

The API named-cert step (§2) is currently manual; it follows the same
create-secret + patch pattern. Run the playbook for the apps side:

```bash
ansible-playbook -i inventory.ini update_apps_cert.yml \
  -e apps_domain=apps.luke.syangsao.net \
  -e acme_home=~/.acme.sh \
  -e ingress_namespace=openshift-ingress \
  -e ingress_controller_namespace=openshift-ingress-operator \
  -e ingress_controller_name=default
```

## Renewal cadence

ZeroSSL DV certs are valid ~90 days. Re-run the ACME issue + apply steps before
expiry (the playbook flags certs expiring within 30 days). After a rebuild, simply
re-issue and re-apply — all steps here are idempotent (`--dry-run=client | oc
apply -f -` and merge-patch).
