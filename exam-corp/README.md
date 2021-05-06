# Anthos Configuration Management Directory

This is the root directory for Anthos Configuration Management.

See [our documentation](https://cloud.google.com/anthos-config-management/docs/repo) for how to use each subdirectory.

----Download the latest version of the Operator CustomResourceDefinition from Google Cloud Storage and apply this to each cluster.#####
gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml config-management-operator.yaml

Install Configuration Management Operator
To configure the behavior of Config Sync, create a configuration file for the ConfigManagement CustomResource, then apply it using the kubectl apply command:
kubectl apply -f config-management-operator.yaml

-----Verify the creation of the operator----
kubectl describe crds configmanagements.configmanagement.gke.io

Google ships a nifty utility called nomos that can be used to manage ACM. Download and add it to the path.
gsutil cp gs://config-management-release/released/latest/linux_amd64/nomos nomos
cp ./nomos /usr/local/bin
chmod +x /usr/local/bin/nomos

----Create an SSH key pair to allow Config Sync to authenticate to bitbucket repository---
openssl genrsa -out private-key.pem 2048
openssl rsa -in private-key.pem -pubout -out public-key.pem


kubectl create secret generic git-creds --namespace=config-management-system --from-file=ssh=PATH_TO_PRIVATE_KEY
kubectl --namespace=config-management-system get secret git-creds -o yaml


----Install ACM----
cat <<EOF | kubectl apply -f -
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  # clusterName is required and must be unique among all managed clusters
  clusterName: <CLUSTER_NAME>
  git:
    syncRepo: git@bitbucket.org:ajayn_sidgs/gcp-acm-lab2.git
    syncBranch: master
    secretType: ssh
    policyDir: "exam-corp"
	syncWait: 2
  # Set to true to install and enable Policy Controller
  policyController:
    enabled: true
    # Uncomment to prevent the template library from being installed
    # templateLibraryInstalled: false
    # Uncomment to disable audit, adjust value to set audit interval
    # auditIntervalSeconds: 0
	# Uncomment to enable support for referential constraints
	referentialRulesEnabled: true
EOF

kubectl -n config-management-system describe configmanagements.configmanagement.gke.io config-management


----Install ACM Constraint----
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
EOF

----Test----
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged-allowed
  labels:
    app: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: false
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged-disallowed
  labels:
    app: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
EOF

kubectl describe k8spspprivilegedcontainer psp-privileged-container
kubectl delete k8spspprivilegedcontainer psp-privileged-container
