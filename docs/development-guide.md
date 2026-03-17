# Development Guide 

This doc explains how to build and run the OnlineBoutique source code locally using the `skaffold` command-line tool.  

## Prerequisites 

- [Docker for Desktop](https://www.docker.com/products/docker-desktop).
- kubectl (can be installed via `gcloud components install kubectl`)
- [skaffold **2.0+**](https://skaffold.dev/docs/install/) (latest version recommended), a tool that builds and deploys Docker images in bulk. 
- A Google Cloud Project with Artifact Registry enabled. 
- Enable GCP APIs for Cloud Monitoring, Tracing, Profiler:
```
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    cloudprofiler.googleapis.com
```
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) (optional - see Local Cluster)
- [Kind](https://kind.sigs.k8s.io/) (optional - see Local Cluster)

## Option 1: Google Kubernetes Engine (GKE)

> 💡 Recommended if you're using Google Cloud Platform and want to try it on
> a realistic cluster. **Note**: If your cluster has Workload Identity enabled, 
> [see these instructions](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#enable)

1.  Create or select a Google Kubernetes Engine cluster and make sure `kubectl`
    is pointing to it:

    ```sh
    export PROJECT_ID="your-project-id"
    export REGION="asia-northeast1"
    export CLUSTER_NAME="swagstore"

    gcloud services enable container.googleapis.com artifactregistry.googleapis.com
    gcloud container clusters get-credentials "${CLUSTER_NAME}" --region "${REGION}" --project "${PROJECT_ID}"
    kubectl get nodes
    ```

2.  Create a Docker repository in Artifact Registry and authenticate Docker:

    ```sh
    export REPOSITORY="swagstore"

    gcloud artifacts repositories create "${REPOSITORY}" \
      --repository-format=docker \
      --location="${REGION}" \
      --description="Swagstore images" || true

    gcloud auth configure-docker "${REGION}-docker.pkg.dev" -q
    export DEFAULT_REPO="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}"
    ```

3.  Install Datadog with Helm before deploying the application. The checked-in
    override file `datadog/values.gke.yaml` expects a secret named
    `datadog-keys` in namespace `datadog` with `api-key` and `app-key` entries.

    ```sh
    kubectl create namespace datadog --dry-run=client -o yaml | kubectl apply -f -
    kubectl create secret generic datadog-keys \
      --namespace datadog \
      --from-literal api-key="${DATADOG_API_KEY}" \
      --from-literal app-key="${DATADOG_APP_KEY}" \
      --dry-run=client -o yaml | kubectl apply -f -

    helm repo add datadog https://helm.datadoghq.com
    helm repo update
    helm upgrade --install datadog-agent datadog/datadog \
      --namespace datadog \
      --create-namespace \
      -f datadog/values.gke.yaml
    ```

4.  Deploy the application with Skaffold. On Apple Silicon, keep
    `--platform=linux/amd64` for GKE-compatible images.

    ```sh
    skaffold run --default-repo="${DEFAULT_REPO}" --platform=linux/amd64
    ```

    This command:

    - builds the container images
    - pushes them to Artifact Registry
    - applies the `./kubernetes-manifests` deployment to Kubernetes

5.  Find the external IP for the frontend:

    ```sh
    kubectl get service frontend-external
    ```


## Option 2 - Local Cluster 

1. Launch a local Kubernetes cluster with one of the following tools:

    - To launch **Minikube** (tested with Ubuntu Linux). Please, ensure that the
       local Kubernetes cluster has at least:
        - 4 CPUs
        - 4.0 GiB memory
        - 32 GB disk space

      ```shell
      minikube start --cpus=4 --memory 4096 --disk-size 32g
      ```

    - To launch **Docker for Desktop** (tested with Mac/Windows). Go to Preferences:
        - choose “Enable Kubernetes”,
        - set CPUs to at least 3, and Memory to at least 6.0 GiB
        - on the "Disk" tab, set at least 32 GB disk space

    - To launch a **Kind** cluster:

      ```shell
      kind create cluster
      ```

2. Run `kubectl get nodes` to verify you're connected to the respective control plane.

3. Run `skaffold run` (first time will be slow, it can take ~20 minutes).
   This will build and deploy the application. If you need to rebuild the images
   automatically as you refactor the code, run `skaffold dev` command.

4. Run `kubectl get pods` to verify the Pods are ready and running.

5. Run `kubectl port-forward deployment/frontend 8080:8080` to forward a port to the frontend service.

6. Navigate to `localhost:8080` to access the web frontend.


## Cleanup

If you've deployed the application with `skaffold run` command, you can run
`skaffold delete` to clean up the deployed resources.
