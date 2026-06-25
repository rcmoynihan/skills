---
name: infra-access
description: How to reach deployed staging/production resources from the shell — kubectl (list/inspect pods, logs, exec), port-forwarding to in-cluster Postgres/Kafka/other services and connecting via psql or a local client, the AWS CLI for AWS services, and the snow CLI for Snowflake. Use when the user wants to query, inspect, or debug a deployed database, running pod, Kafka topic, cloud resource, or any service running in a cluster or cloud environment.
---

# Accessing deployed infrastructure

Patterns for inspecting or querying resources running in staging or production — Postgres, running
pods, Kafka, and cloud services (AWS, Snowflake).

## Guardrails

- **Staging and read-only by default.** Unless the user indicates otherwise, target staging and use
  inspection/read-only commands only.
- **Production is opt-in, then sticky.** If the user names production, or asks for live production
  debugging / questions / actions, treat that as authorization to work against production for the
  rest of that line of work — do **not** re-ask before each command. Likewise, once the user asks
  for an *action* (not just a read), you don't need to confirm every subsequent step.
- **Announce destructive operations.** Clearly irreversible operations (`DROP`/`DELETE`/`TRUNCATE`,
  deleting a topic, terminating an instance) should be stated plainly before you run them — say what
  you're about to do, then proceed; don't bury them.
- **Clean up tunnels.** Run port-forwards in the background and kill them when you're done.

## Figure out the environment; don't assume

There are several clusters, namespaces, AWS profiles, and Snowflake connections. Discover the right
one rather than guessing:

```bash
kubectl config get-contexts        # available clusters — pick staging vs prod by name
kubectl get namespaces             # find the namespace for the service
aws configure list-profiles        # available AWS profiles/accounts
snow connection list               # configured Snowflake connections
```

The repo itself often pins the answer — check Helm charts, `k8s/`/`deploy/` manifests, `.env`
files, and READMEs for the service's namespace, database name, and service names.

## Kubernetes

```bash
kubectl --context <ctx> -n <ns> get pods
kubectl --context <ctx> -n <ns> logs <pod> [-f] [--previous]
kubectl --context <ctx> -n <ns> describe pod <pod>
kubectl --context <ctx> -n <ns> exec -it <pod> -- sh
```

## Port-forward to in-cluster services

Reach a service that isn't publicly exposed by forwarding it to localhost, then connect with a
local client.

```bash
# Postgres
kubectl --context <ctx> -n <ns> port-forward svc/<pg-svc> 5432:5432 &
psql -h localhost -p 5432 -U <user> <db>
# password is usually in a k8s secret:
kubectl --context <ctx> -n <ns> get secret <secret> -o jsonpath='{.data.password}' | base64 -d

# Kafka
kubectl --context <ctx> -n <ns> port-forward svc/<kafka-svc> 9092:9092 &
# then point a kafka client / kcat at localhost:9092

# Any service
kubectl --context <ctx> -n <ns> port-forward svc/<svc> <local>:<remote> &
```

## AWS

```bash
aws sso login --profile <profile>          # if the profile uses SSO
aws --profile <profile> --region <region> <service> <describe|list|get> ...
```

## Snowflake

```bash
snow sql -c <connection> -q "<read-only query>"
```
