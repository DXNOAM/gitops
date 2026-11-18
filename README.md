# GitOps - Dynamic Generator

This lab implements GitOps with an Argo CD ApplicationSet and Git Directory
Generators. It defines two applications and selects only the `qa` environment
for deployment:

- Flask uses the Helm output that Jenkins publishes to `workloads/qa/flask-aws-monitor/app.yaml`.
- NGINX uses the Deployment in `systems/team-b/qa/app2.yaml`, with two replicas.

## Directory Structure

```text
gitops/
├── README.md
├── manifests/
│   ├── app1.yaml
│   └── app2.yaml
├── applicationsets/
│   └── qa-applicationset.yaml
└── systems/
    ├── devops/
    │   ├── dev/
    │   │   └── ingress.yaml
    │   └── qa/
    │       └── ingress.yaml
    ├── team-a/
    │   ├── dev/
    │   │   └── app1.yaml
    │   ├── prd/
    │   │   └── app1.yaml
    │   └── qa/
    │       └── app1.yaml
    └── team-b/
        ├── dev/
        │   └── app2.yaml
        ├── prd/
        │   └── app2.yaml
        └── qa/
            └── app2.yaml
```

The application files under `systems/` are identical copies of the original files
in `manifests/`. The original files remain unchanged.

## Key Files

| File | Purpose |
| --- | --- |
| [manifests/app1.yaml](manifests/app1.yaml) | Original Flask Application definition |
| [manifests/app2.yaml](manifests/app2.yaml) | Original NGINX Deployment |
| [applicationsets/qa-applicationset.yaml](applicationsets/qa-applicationset.yaml) | Discovers the QA directories and creates the two Applications |
| [systems/team-a/qa/app1.yaml](systems/team-a/qa/app1.yaml) | Copy of the Flask definition documenting the lab structure |
| [systems/team-b/qa/app2.yaml](systems/team-b/qa/app2.yaml) | Deployment used to deploy NGINX in QA |

## Exactly Two QA Applications

A single ApplicationSet contains two Git Directory Generators:

| Selected Directory | Generated Application | Deployment Source | Namespace |
| --- | --- | --- | --- |
| `systems/team-a/qa` | `flask-aws-monitor-qa` | `workloads/qa/flask-aws-monitor` in this repository | `flask-aws-monitor-qa` |
| `systems/team-b/qa` | `my-nginx-app-qa` | `systems/team-b/qa` in the GitOps repository | `my-nginx-app-qa` |

Only the two QA directories are selected. The `dev`, `prd`, and `devops`
directories are excluded from selection, so the ApplicationSet creates exactly
two Applications with the current structure. The generators discover the
directories in Git and use `values` to pass each application's source settings
to the shared template.

The paths are explicitly configured as `systems/team-a/qa` and
`systems/team-b/qa`. Adding another team under `systems/` requires adding a
corresponding generator; the current configuration does not automatically select
QA directories belonging to additional teams.

### Flask

The ApplicationSet discovers Flask through `systems/team-a/qa` and creates
`flask-aws-monitor-qa`. It reads the workload from `main` in this repository
at `workloads/qa/flask-aws-monitor`. Jenkins renders and publishes `app.yaml`
there after pushing the image with the same build tag to Docker Hub.

The original `manifests/app1.yaml` and its copies under `systems/team-a/`
document the lab structure. Manage the QA workload through `systems-qa`.

### NGINX

NGINX uses the original Deployment, including the `nginx:latest` image, two
replicas, and the `app: nginx` label. Its Application uses the built-in `default`
project. Both Application names and destination namespaces use the `qa` suffix
to separate the QA environment.

### Synchronization Policy

- Automated synchronization applies changes from each application's Git source.
- `prune: true` removes resources deleted from the deployment source.
- `selfHeal: true` corrects drift between the cluster state and the Git definition.
- `CreateNamespace=true` creates the destination namespace when needed.

## Ingress

To retain the layout shown in the lab reference, each
`systems/devops/{dev,qa}/ingress.yaml` contains an Ingress and a Service that
selects NGINX using its existing `app: nginx` label. These files contain no
additional Deployment and are not selected by the ApplicationSet.

If you use an Ingress file separately in the future, deploy it in the same
namespace as NGINX, install an Ingress controller supporting the `nginx` class,
and configure the hostname to resolve to the controller. You can also check
NGINX without an Ingress or Service by forwarding a port to its Deployment.

## Prerequisites for Future Deployment

- Kubernetes and Argo CD with the ApplicationSet controller in the `argocd` namespace.
- Argo CD read access to this GitOps repository.
- The `default` AppProject permits this repository and deployment to both QA namespaces.
- A successful Jenkins build of the application `main` branch has published
  `workloads/qa/flask-aws-monitor/app.yaml` to this repository's `main` branch.
  Configure Jenkins using the [application README](https://github.com/DXNOAM/devops-course-app-June-2026#jenkins-setup-and-stages).

Publish the first Flask manifest before applying the updated ApplicationSet,
so Argo CD can already read the new source path.

## Future Deployment and Verification

After an explicit decision to deploy and once the prerequisites are met, run
this installation command from the repository root:

```powershell
kubectl apply -f applicationsets/qa-applicationset.yaml
```

After deployment, verify the generated Applications and workloads:

```powershell
kubectl get applicationsets systems-qa -n argocd
kubectl get applications -n argocd -l environment=qa
kubectl get deployments -n flask-aws-monitor-qa
kubectl get deployment my-nginx-app -n my-nginx-app-qa
kubectl port-forward -n my-nginx-app-qa deployment/my-nginx-app 8080:80
```

While port forwarding is active, open `http://localhost:8080` to check NGINX.

Do not manually apply `manifests/app1.yaml` or its copies alongside the
ApplicationSet, since doing so would create an additional, separate Flask
Application.

## Local Validation Completed

- Parsed the YAML files using `kubectl patch --local`, without contacting a cluster API.
- Compared all six application copies under `systems/` with the originals; their contents are identical.
- Checked directory selection and local template parameter substitution; the result contains exactly
  `flask-aws-monitor-qa` and `my-nginx-app-qa`.
- Checked that the optional Service matches the existing NGINX labels and port.

These checks did not run the ApplicationSet controller, validate against the CRD
schema, or verify Flask availability in a cluster.

## Reference

[Argo CD Git Directory Generator documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/)
