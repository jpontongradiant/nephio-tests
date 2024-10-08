cat << EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.28.9
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF

kpt live apply gitea --reconcile-timeout 15m --output=table
kpt live apply network-config --reconcile-timeout=15m --output=table
kpt live apply resource-backend --reconcile-timeout=15m --output=table
kpt live apply gitea --reconcile-timeout 15m --output=table

kpt live apply porch --reconcile-timeout=15m --output=table
kpt live apply nephio-operator --reconcile-timeout=15m --output=table

kubectl apply -f  - <<EOF
apiVersion: v1
kind: Secret
metadata:
    name: git-user-secret
    namespace: nephio-system
type: kubernetes.io/basic-auth
stringData:
    username: nephio
    password: secret
EOF

kpt live apply configsync --reconcile-timeout=15m --output=table
kpt live apply stock-repos --reconcile-timeout=15m --output=table
kpt live apply nephio-webui --reconcile-timeout=15m --output=table

kpt live apply cert-manager --reconcile-timeout 15m --output=table
kpt live apply cluster-capi --reconcile-timeout 15m --output=table
kpt live apply cluster-capi-infrastructure-docker --reconcile-timeout 15m --output=table
kpt live apply cluster-capi-kind-docker-templates --reconcile-timeout 15m --output=table

kpt live apply configsync --reconcile-timeout=15m --output=table

#kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/repository@@origin/v3.0.0 mgmt
kpt fn render mgmt
kpt live init mgmt
kpt live apply mgmt --reconcile-timeout=15m --output=table


kpt live apply rootsync --reconcile-timeout=15m --output=table
kubectl get rootsyncs.configsync.gke.io -A -w




porchctl rpkg get --name nephio-workload-cluster
porchctl rpkg clone -n default catalog-infra-capi-03f91491466438cfe5dc1f5ad92d48091635dd96 --repository mgmt regional
porchctl rpkg pull -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
kpt fn eval --image gcr.io/kpt-fn/set-labels:v0.2.0 regional -- "nephio.org/site-type=regional" "nephio.org/region=us-west1"
porchctl rpkg push -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
porchctl rpkg propose -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
porchctl rpkg approve -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
kubectl get cl -w


kubectl get secret regional-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/regional-kubeconfig
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/regional-kubeconfig

kubectl get ns --context regional-admin@regional
kubectl get machinesets
k --context regional-admin@regional apply -f calico.yaml 
k --context regional-admin@regional get pods -A
kubectl get machinesets

kubectl apply -f test-infra/e2e/tests/free5gc/002-edge-clusters.yaml
kubectl get machinesets

