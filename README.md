# csdp-salesdemo-clusters
Cluster definitions (eksctl) for csdp-salesdemo clusters

## Cluster Setup
**NOTE**: The cluster you create will run on spot instances, which may get cycled from time to time. This means that the environment _could_ become unstable for short periods of time
1. `export cluster_name=<your-cluster-name>; export environment=<your-directory>`
1. determine your cluster name and environment  (`${cluster_name}` and `${environment}`)
1. S3
    1. export your creds and `AWS_DEFAULT_REGION`
    1. create s3 bucket with your cluster name (`aws s3api create-bucket --bucket  ${cluster_name} --region ${cluster_region}`)
    1. Create a lifecycle action on the bucket to remove old data (`aws s3api put-bucket-lifecycle --bucket ${cluster_name} --lifecycle-configuration file://_template/s3-lifecycle.json`)
1. eksctl
    1. copy the `_template/cluster-config.yaml` replacing `${cluster_name}` with your cluster's name
    1. `eksctl create cluster -f ${environment}/cluster-config.yaml --kubeconfig ~/.kube/config.${cluster_name}`
1. aws certmanager
    1. Use aws certmanager to create a cert for this dns entry, validate with DNS, and add the record to Route53 from the cert manager
    1. Copy the cert's ARN
1. kubectl/helm
    1. `export KUBECONFIG=~/.kube/config.${cluster_name}`
    1. Copy and update `_template/ingress-nginx-values.yaml` replacing the cert arn
    1. Add the nginx helm repo if you dont have it `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`
    1. `helm install ingress-nginx ingress-nginx/ingress-nginx -f ${environment}/ingress-nginx-values.yaml -n ingress-nginx --create-namespace` (https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)
1. route53
    1. Create route53 entry (I am using the hosted zone `csdp-salesdemo.sa.cf-cd.com` for these cluster(s)). Load balancer is `kubectl get svc -n ingress-nginx ingress-nginx-controller -ojsonpath='{.status.loadBalancer.ingress[0].hostname}'`
    1. point route53 entry to loadbalancer that was created


_todo/wishlist_: ExternalDNS, certmanager, automate bucket creation, eksctl with fargate spot, sealedsecrets, (further future) see if can bootstrap tools with argocd boostrap, convert res definitions to crossplane

# Runtime setup
1. Download the latest runtime (for mac: `curl -L --output - https://github.com/codefresh-io/cli-v2/releases/latest/download/cf-darwin-amd64.tar.gz | tar zx && mv ./cf-darwin-amd64 /usr/local/bin/cf && cf version`)
1. `export ingress_host=<>.<>.sa.cf-cd.com`
1. Run the installer: `cf runtime install ${cluster_name} --ingress-host https://${ingress_host} --git-src-provider github --git-src-repo https://github.com/codefresh-contrib/${cluster_name}`
1. if You install correctly but get `FATAL failed to create default git integration: failed to add git integration: failed making a graphql API call while adding git integration: 404 Not Found`, try this: `cf integration git add default --provider github --runtime ${cluster-name} --api-url https://api.github.com`

1. Add secrets from `_template/secrets.yaml` for cluster

1. s3 logging and artifacts
    1. Update `apps/workflows/overlays/${cluster_name}/kustomization.yaml` in your bootstrap repo with `installer-patch.yaml`. This will write logs to s3 and make s3 available for artifacts.
    1. If you want your artifacts stored separately from your logs, Add `artifacts-repo.yaml` to `apps/workflows/overlays/${cluster_name}/`, and add `- artifacts-repo.yaml` to the resources defined in `installer-patch.yaml`

# Add an applications repository
1. ...
1.  If you need different creds than what you gave autopilots, follow this guide to update the configmap (argocd-cm): https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repository-credentials
