# Knative and Private Images without Pull Secrets

This example is applicable if you want to allow images to be pulled without creating a kubernetes secret in every namespace.

This assumes you assume you trust the kubelet to pull any images by providing the auth to it.

The problem to solve is that the Knative Controller needs access to the image at the registry to be able to resolve the image
provided in the knative service definition to a sha256 digest, if you provide the image url with a sha256 then the controller doesn't
need access to the regisgtry.

The reason for this is that in Knative enforce a good practice that images should be reference by digest for the revision, making
every revision an inmutable entity.

In normal cases, the user will create a pull secret for the service account to be use by the Knative service, the Controller is granted access to all namespaces secrets to be able access this pull secrets and use it to resove the image.

In this example we want to avoid creating the pull secrets for every service account across multiple namespaces. and just provide the Controller with the secrets once, similar on what we did with the kubelet.


## Prerequisites

Install [Kind](https://kind.sigs.k8s.io/)

## Setup Kind Cluster

```bash
mkdir $HOME/.config/kind/
cat > $HOME/.config/kind/cluster.yaml << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.19.4
  extraPortMappings:
  - containerPort: 31080 # expose port 31380 of the node to port 80 on the host, later to be use by kourier ingress
    hostPort: 80
  extraMounts:
  - containerPath: /var/lib/kubelet/config.json
    hostPath: $HOME/.config/kind/docker-config.json
EOF
```

## Create GCP service account

Set Registry Namespace and Project
```bash
PROJECT_ID=<redacted>
REGISTRY="gcr.io/${PROJECT_ID}"
```

```bash
SA_NAME=kubernetes-kind
SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Set project to own the service account
gcloud config set project "${PROJECT_ID}"

# Create GCP service account
gcloud iam service-accounts --format='value(email)' create "${SA_NAME}"

# Grant service account permission to read from GCR
gsutil iam ch "serviceAccount:${SA_EMAIL}:objectViewer" "gs://artifacts.${PROJECT_ID}.appspot.com"
```

## Create Docker Auth Config (Ubuntu)
```bash
DOCKER_CONFIG=$(mktemp -d)
export DOCKER_CONFIG

# Generate SA Credentials
gcloud iam service-accounts keys create "${DOCKER_CONFIG}/key.json" --iam-account "${SA_EMAIL}"

# Log into GCR
cat "${DOCKER_CONFIG}/key.json" | docker login -u _json_key --password-stdin https://gcr.io

# Copy config to kind bind-mount path
cp "${DOCKER_CONFIG}/config.json" "$HOME/.config/kind/docker-config.json"

rm -r "${DOCKER_CONFIG}"
unset DOCKER_CONFIG
```

### Create Docker Auth Config (Mac)
```bash
DOCKER_CONFIG=$(mktemp -d)

# Generate Credentials
gcloud iam service-accounts keys create "${DOCKER_CONFIG}/key.json" --iam-account=$SA_EMAIL

# Generate Docker Config
cat > $HOME/.config/kind/docker-config.json << EOF
{
  "auths": {
    "gcr.io": {
      "auth": "$((echo -n "_json_key:" && cat "${DOCKER_CONFIG}/key.json") | base64)"
    }
  },
  "HttpHeaders": {
    "User-Agent": "Docker-Client/19.03.5 (linux)"
  }
}
EOF

rm -r $DOCKER_CONFIG
unset DOCKER_CONFIG
```

## Create kind Cluster
```bash
kind create cluster --name knative --config $HOME/.config/kind/cluster.yaml
```

## Install Knative Serving on Kind
```bash
curl -sL https://raw.githubusercontent.com/csantanapr/knative-kind/master/02-serving.sh | bash
```

## Build Knative Go Sample
```bash
git clone --depth 1 https://github.com/knative/docs knative-docs
pushd knative-docs/docs/serving/samples/hello-world/helloworld-go
gcloud auth login
gcloud auth configure-docker
docker build -t "${REGISTRY}/helloworld-go" .
docker push "${REGISTRY}/helloworld-go"
```

Now deploy the Knative service using image `${REGISTRY}/helloworld-go`
```bash
kubectl apply -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: ${REGISTRY}/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative"
EOF

```

### Get the status of the Knative service
```bash
kubectl get ksvc hello
```
There is a problem with the revision `RevisionFailed`
```
NAME    URL                                     LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.127.0.0.1.nip.io   hello-00002     hello-00001   False   RevisionFailed
```

describe to get more info

```bash
kubectl describe ksvc hello
```
Lets check out the status
```
Status:
  Address:
    URL:  http://hello.default.svc.cluster.local
  Conditions:
    Last Transition Time:        2020-12-04T03:15:53Z
    Message:                     Revision "hello-00002" failed with message: Unable to fetch image "gcr.io/${PROJECT_ID}/helloworld-go": failed to resolve image to digest: HEAD https://gcr.io/v2/${PROJECT_ID}/helloworld-go/manifests/latest: unsupported status code 401.
    Reason:                      RevisionFailed
    Status:                      False
    Type:                        ConfigurationsReady
    Last Transition Time:        2020-12-04T03:15:53Z
    Message:                     Revision "hello-00002" failed with message: Unable to fetch image "gcr.io/${PROJECT_ID}/helloworld-go": failed to resolve image to digest: HEAD https://gcr.io/v2/${PROJECT_ID}/helloworld-go/manifests/latest: unsupported status code 401.
    Reason:                      RevisionFailed
    Status:                      False
    Type:                        Ready
    Last Transition Time:        2020-12-04T02:06:05Z
    Status:                      True
    Type:                        RoutesReady
  Latest Created Revision Name:  hello-00002
  Latest Ready Revision Name:    hello-00001
  Observed Generation:           2
```

Since we indicated the image url in the yaml without a digest, by default the controller needs access to resolve the image url to an url that includes the digest. It does a `HEAD` http request to get it but it needs auth.

## Fix 1. Disable Tag resolution
One way to fix is to disabele tag resolution like indicated in the doc [skipping-tag-resolution](https://knative.dev/docs/serving/tag-resolution/#skipping-tag-resolution)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-deployment
  namespace: knative-serving

data:
  # List of repositories for which tag to digest resolving should be skipped
  registriesSkippingTagResolving: gcr.io
```



## Fix 2. Provide DIGEST with Image URL
One way to fix this is to provide the image url with the digest

Lets delete the ksvc
```bash
kubectl delete ksvc hello
```
Lets the get the digest from the registry, go to the console and get the value
```bash
DIGEST=sha256:d5d36bca4af1ad2f7a05768f96e0fc703e24c0057c30a3e5aa019953f88dd1f9
```

```bash
kubectl apply -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: ${REGISTRY}/helloworld-go@${DIGEST}
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative"
EOF

```

Now list the service again
```bash
kubectl get ksvc hello
```
Notice that the Ready column is `True`
```
NAME    URL                                     LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.127.0.0.1.nip.io   hello-00001     hello-00001   True
```
Git it a try
```
curl http://hello.default.127.0.0.1.nip.io
```
```
Hello Knative!
```

This works but is going to be `PITA` to tell all your friends that are going to use your cluster to specify the image with sha256.
So lets give the Knative Controller access to the authentication file we already gave the kubelet.
## Fix 3. Providing DockerConfig to Knative Controller
The Knative Controller leverage the project [k8schain](https://github.com/google/go-containerregistry/tree/master/pkg/authn/k8schain) to figure out how to authenticate with the registry reference by the image url.
The k8schain at some point was part of the kubernetes kubelet and recently got remove, and now the go-containerregistry has a fork of this code https://github.com/vdemeester/k8s-pkg-credentialprovider

The `k8s-pkg-credentialprovider` offers support for different registries and also default docker config file.
In our case the controller would not be running in a VM on GCP would not be able to take advantage of the GCP metatada service stated here https://github.com/vdemeester/k8s-pkg-credentialprovider/tree/master/gcp

So lets take advantage of docker config file, we need to provide the file to the controller in three different places at `workingDirPath` `./config.json`, `homeJsonDirPath` `$HOME/.docker/config.json` , `rootJsonDirPath` `/root/.docker/config.json` in that order of preference.
https://github.com/vdemeester/k8s-pkg-credentialprovider/blob/master/config.go#L91

One option is that you can modify the controller deployment, and mount the hostpath that is use for the kubelet, but you would need to give certain permission to the controller to access.

Another option is you can create a secret and mount this secret on the controller.

Create the secret `container-registry` in the namespace of the Controller `knative-serving`
```bash
kubectl create secret generic container-registry -n knative-serving \
--from-file=config.json=$HOME/.config/kind/docker-config.json
```
NOTE: Do not use file name `.dockerconfigjson` use `config.json`

Edit the deployment for the controller
```bash
kubectl edit -n knative-serving deployment controller
```

Edit Pod template to add the volume and mount
```yaml
volumeMounts:
- name: container-registry
  mountPath: ".docker"
volumes:
- name: container-registry
  secret:
    secretName: container-registry
```

Now re-create the Knative Service without a digest
```bash
kubectl delete ksvc hello
```

Now deploy the Knative service using image `${REGISTRY}/helloworld-go`
```bash
kubectl apply -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: ${REGISTRY}/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative"
EOF

```


Now list the service again
```bash
kubectl get ksvc hello
```
Notice that the Ready column is `True`
```
NAME    URL                                     LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.127.0.0.1.nip.io   hello-00001     hello-00001   True
```
Git it a try
```bash
curl http://hello.default.127.0.0.1.nip.io
```
```
Hello Knative!
```

To double check you can create a new image with a different tag `2.0`

Change the `helloword.go`
```
fmt.Fprintf(w, "Hello Private Image\n")
```

Build the image with tag `2.0`
```
docker build -t "${REGISTRY}/helloworld-go:2.0" .
docker push "${REGISTRY}/helloworld-go:2.0"
```

This time lets use the `kn` CLI to update the service
```bash
kn service update hello --image "${REGISTRY}/helloworld-go:2.0"
```

Let's test the service
```bash
curl http://hello.default.127.0.0.1.nip.io
```
```
Hello Private Image
```