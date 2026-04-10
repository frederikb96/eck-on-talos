# Elastic Stack (ECK on Talos)

> **Template — copy this file into your own private repository as `README.md`** after following [eck-on-talos](https://github.com/frederikb96/eck-on-talos), then fill in the placeholders below. It's a short operator-facing reference, not a tutorial.

## Overview

This repository contains the infrastructure-as-code and operational docs for our internal Elastic Stack, running as an ECK deployment on a 3-node Talos Kubernetes cluster.

- **Cluster nodes:** `node1` (`<IP1>`), `node2` (`<IP2>`), `node3` (`<IP3>`)
- **Kibana:** https://`<KIBANA_DNS_OR_IP>`:30601
- **Elasticsearch HTTP:** https://`<ES_DNS_OR_IP>`:30920
- **Fleet Server:** https://`<FLEET_DNS_OR_IP>`:30822
- **Internal CA:** `<LOCATION_WHERE_CA_KEY_IS_STORED_E_G_PASSWORD_MANAGER>`
- **Stack version:** `<e.g. 9.3.2>`
- **Owner / on-call:** `<TEAM_NAME>`, `<EMAIL_OR_PAGER>`

## Repository layout

```
talos/         Talos machine configs (patches + per-node files)
kubernetes/    StorageClass, PVs, ECK operator values, eck-stack values
ca/            Internal CA (gitignored; real files live in password manager)
```

## Access

- Log in to Kibana with `elastic` + password from `kubectl -n elastic-stack get secret elasticsearch-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'`
- The `elastic` superuser is for break-glass only. Regular users: `<AUTH_MODEL_EG_NATIVE_REALM_OR_SAML>`
- Install the internal CA (`ca/ca.crt`) on your workstation to silence browser warnings. Distribution: `<HOW_YOUR_ORG_DISTRIBUTES_CAS>`

## Day-2 operations

Everything in this repo is declarative. To change anything:

1. Edit the relevant YAML file in a PR
2. Run the commands in the [Maintenance section of the upstream guide](https://github.com/frederikb96/eck-on-talos#maintenance) against the cluster
3. Commit and merge

### Common tasks

- **Upgrade the Elastic Stack version** → bump every `version:` field in `kubernetes/eck-stack/values.yaml` and run `helm upgrade eck-stack elastic/eck-stack --version <chart-version> -n elastic-stack -f kubernetes/eck-stack/values.yaml`
- **Upgrade Talos** → bump `talos_version` below, follow the upstream [Talos upgrade procedure](https://github.com/frederikb96/eck-on-talos#upgrading-talos)
- **Upgrade the ECK operator** → bump the `--version` in the operator `helm upgrade` command from the upstream guide
- **Rotate the internal CA** → follow the upstream [CA rotation procedure](https://github.com/frederikb96/eck-on-talos#rotating-the-internal-ca)
- **Add a Kibana/Fleet user** → Kibana → Stack Management → Users
- **Add an external Elastic Agent** → Kibana → Fleet → Agent policies → copy token, install agent with `--certificate-authorities=ca/ca.crt`

## Known runtime footprint

| Component | Replicas | CPU request | Memory request | Memory limit |
|---|---|---|---|---|
| Elasticsearch | 3 | 500m | 8 GiB | 8 GiB |
| Kibana | 2 | 100m | 1 GiB | 2 GiB |
| Fleet Server | 2 | 50m | 512 MiB | 2 GiB |
| Elastic Agent DS | 3 (one per node) | 100m | 512 MiB | 2 GiB |

## Versions

```
talos_version    = "v1.12.6"
eck_operator     = "3.3.1"
eck_stack_chart  = "0.18.1"
elastic_stack    = "9.3.2"
```

(Update these whenever you bump the corresponding value in the repo.)

## Incident response

1. Check cluster health: `kubectl -n elastic-stack get elasticsearch,kibana,agent`
2. Check pod status: `kubectl -n elastic-stack get pods`
3. Check the Stack Monitoring dashboards in Kibana (they're self-monitoring, so they keep working as long as at least one ES node is up)
4. Talos node issues: `talosctl -n <ip> dmesg -f`, `talosctl -n <ip> logs kubelet`
5. Escalation path: `<DEFINE_YOUR_ESCALATION>`

## References

- Upstream deployment guide: <https://github.com/frederikb96/eck-on-talos>
- ECK docs: <https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html>
- Talos docs: <https://www.talos.dev/latest/>
