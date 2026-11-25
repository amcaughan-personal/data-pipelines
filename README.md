# data-pipelines

Data pipelines that run on Airflow on the shared EKS cluster from
[eks-platform](https://github.com/amcaughan-personal/eks-platform), as its first tenant team.
Each pipeline pulls a feed from the [data-simulator API](https://github.com/amcaughan-personal/data-simulator-api),
lands it in S3 and models it with dbt on Athena. Every Airflow task runs as its own pod, so
work scales onto spot nodes and back to nothing.

**Status: not started.** This README is the plan.

## How it fits with eks-platform

- **eks-platform** runs the cluster, Airflow itself and the shared add-ons, and onboards this
  repo as a tenant with one file, `gitops/tenants/data-pipelines/values.yaml`: a namespace
  with a budget and default pod sizes, an Argo CD project limited to this repo's `k8s/`
  folder and that namespace, traffic into the namespace only from inside it, and Pod
  Identity links from its service accounts to this repo's roles.
- **Pod roles** must be named `data-pipelines-pod-<name>` (this repo's CI already creates
  roles only with its own prefix and permissions boundary). eks-platform's CI role may hand
  any such role to pods, so no personal-infra change is needed per role.
- **Each pod role trusts only its own pods.** Pod Identity tags every credential request with
  the cluster, namespace and service account; each role's trust policy requires
  `kubernetes-namespace = data-pipelines` and its own `kubernetes-service-account`, so the role
  refuses any other pods, even ones the platform linked it to by mistake.
- **This repo** owns the work: pipeline definitions, DAGs, container images, and its own AWS
  resources (buckets, Glue databases, the IAM roles its pods use).
- Airflow pulls the DAGs from here with git-sync; anything else this repo runs on the cluster
  goes in `k8s/`, which Argo CD deploys into this repo's namespace.

## Planned layout

```
pipelines/     one file per pipeline: its API spec, schedule and dbt models
dags/          one DAG generator reading pipelines/
ingest/        one container image: an API spec in, raw JSON lines in S3 out
dbt/           one dbt project, models/<pipeline>/
k8s/           what Argo CD deploys into this repo's namespace
infra/         S3 buckets, Glue databases, pod roles named data-pipelines-pod-* (Terragrunt)
```

## To do

### Foundations
- [ ] Terragrunt layout like the other repos (`root.hcl`, `.tool-versions`, CI plan, drift check)
- [ ] personal-infra: plan and apply roles for this repo, its `infra-*` environments, required checks
- [ ] ECR repositories for the `ingest` and `dbt` images, and CI that builds and pushes them
- [ ] `infra/`: the raw bucket, Glue databases, and the pod role `data-pipelines-pod-tasks` (S3, Glue, Athena), trusting only `pods.eks.amazonaws.com` with `kubernetes-namespace = data-pipelines` and `kubernetes-service-account = tasks`

### The first pipeline (payments, hourly)
- [ ] `pipelines/payments.yaml`: the API spec, schedule and models
- [ ] `ingest/`: pull an hour's window from the API and write it to S3 (tested locally against the API)
- [ ] `dbt/`: bronze, silver and gold models for payments (run locally against Athena)
- [ ] `dags/`: generate a DAG per pipeline; ingest, then dbt, as KubernetesPodOperator tasks

### On the cluster
- [ ] eks-platform: Airflow on Argo CD (KubernetesExecutor, git-sync from `dags/`, Postgres, task logs to S3), built on kind first
- [ ] Decide where task pods run: in this repo's namespace (so they count against its budget and get its pod roles), via Airflow's pod template
- [ ] Decide how Airflow reads live task logs: the tenant's network policy blocks traffic into the namespace from Airflow's, so either the tenant chart allows Airflow's namespace in, or logs come only from S3
- [ ] eks-platform: onboard this repo: `gitops/tenants/data-pipelines/values.yaml` with its repo, `k8s` path, budget, and `podRoles` (e.g. `tasks: data-pipelines-pod-tasks`)
- [ ] Run payments end to end on kind, then once on EKS

### More pipelines
- [ ] IoT sensors every 5 minutes (anomalies), inventory daily
- [ ] A gold model fed by two pipelines, run when both update (Airflow Assets)
- [ ] Dashboards and alerts for this repo's pipelines (rows per hour, last successful dbt run)
- [ ] Design decisions doc
