# jupyter server with custom image

```
python components/build_image.py --tf_version=2.1.0 --platform=cpu tf_notebook
docker push <repository> # of the above image (use: docker images)

# go to kubeflow ui running on http://127.0.0.1:8080
# create a new jupyter server, use the above repository
```

# kind cluster

```
# Optional: delete cluster if exists
kind delete cluster --name tr8ai

# checkout tr8ai kubeflow (root here is /tr8ai)
mkdir /tr8ai
cd /tr8ai
git clone https://github.com/tr8-ai/kubeflow.git
cd kubeflow/tr8ai_kind

# currently the kind manifest creates 1 controller + 2 workers using k8s 1.19.3
# mounts the local /scratch directory to controller and workers, create /scratch, /scratch/worker0, /scratch/worker1
mkdir /scratch
mkdir /scratch/worker0
mkdir /scratch/worker1

# create kind cluster with name tr8ai (k8s-cluster-name will be kind-tr8ai): 
kind create cluster --config=tr8ai_kind.yaml --name tr8ai

# Deploy kubeflow
# Most recent kfdef v1.2.0
# Using /scratch/kubeflow_deployments folder 
mkdir -p /scratch/kubeflow_deployments/kubeflow
cd /scratch/kubeflow_deployments/kubeflow
kfctl build -V -f https://raw.githubusercontent.com/kubeflow/manifests/master/kfdef/kfctl_k8s_istio.v1.0.2.yaml
# above is handy just grabs the files, for mods and inspection
# deploy the yaml
kfctl apply -V -f kfctl_k8s_istio.v1.0.2.yaml


# Inject gcr.io access to nodes 
cd /tr8ai/kubeflow/tr8ai_kind
bash kind-gcr.sh

```

# kf-pipelines running in kind

```
export PIPELINE_VERSION=1.0.4

# remove existing
kubectl delete -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic-pns?ref=$PIPELINE_VERSION"
kubectl delete -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"

# this will get stuck due to terminating 

kubectl get namespace kubeflow  -o json > tmp.json

# change the finalizing condition
# FROM
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },

# TO
    "spec": {
        "finalizers": []
    },

# start proxy and apply the finalizer change
kubectl proxy &
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/kubeflow/finalize


# deploy
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic-pns?ref=$PIPELINE_VERSION"

```
