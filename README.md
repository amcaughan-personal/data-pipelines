# data-pipelines

Data pipelines that run on Airflow on the shared EKS cluster from
[eks-platform](https://github.com/amcaughan-personal/eks-platform), as its first tenant team.
Each pipeline pulls a feed from the [data-simulator API](https://github.com/amcaughan-personal/data-simulator-api),
lands it in S3 and models it with dbt on Athena. Every Airflow task runs as its own pod, so
work scales onto spot nodes and back to nothing.

**Status: not started.** This README is the plan.

## How it fits with eks-platform

- **eks-platform** runs the cluster, Airflow itself and the shared add-ons, and onboards this
  repo as a tenant: a namespace with a budget, an Argo CD project limited to this repo, and
  Pod Identity links for this repo's pod roles (`gitops/tenants/data-pipelines/`).
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
- [ ] `infra/`: the raw bucket, Glue databases, and the pod role `data-pipelines-pod-tasks` (S3, Glue, Athena)

### The first pipeline (payments, hourly)
- [ ] `pipelines/payments.yaml`: the API spec, schedule and models
- [ ] `ingest/`: pull an hour's window from the API and write it to S3 (tested locally against the API)
- [ ] `dbt/`: bronze, silver and gold models for payments (run locally against Athena)
- [ ] `dags/`: generate a DAG per pipeline; ingest, then dbt, as KubernetesPodOperator tasks

### On the cluster
- [ ] eks-platform: Airflow on Argo CD (KubernetesExecutor, git-sync from `dags/`, Postgres, task logs to S3), built on kind first
- [ ] eks-platform: onboard this repo (`gitops/tenants/data-pipelines/values.yaml`)
- [ ] Run payments end to end on kind, then once on EKS

### More pipelines
- [ ] IoT sensors every 5 minutes (anomalies), inventory daily
- [ ] A gold model fed by two pipelines, run when both update (Airflow Assets)
- [ ] Dashboards and alerts for this repo's pipelines (rows per hour, last successful dbt run)
- [ ] Design decisions doc
