# Deploy hashicorp Vault on minikube with TLS

### Init 
export VAULT_K8S_NAMESPACE="vault" \
export VAULT_HELM_RELEASE_NAME="vault" \
export VAULT_SERVICE_NAME="vault-internal" \
export K8S_CLUSTER_NAME="cluster.local" \
export WORKDIR=.

### Create the certificate

openssl genrsa -out ${WORKDIR}/vault.key 2048

openssl req -new -key ${WORKDIR}/vault.key -out ${WORKDIR}/vault.csr -config ${WORKDIR}/vault-csr.conf

### Issue the Certificate.

cat > ${WORKDIR}/csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
   name: vault.svc
spec:
   signerName: kubernetes.io/kubelet-serving
   expirationSeconds: 8640000
   request: $(cat ${WORKDIR}/vault.csr|base64|tr -d '\n')
   usages:
   - digital signature
   - key encipherment
   - server auth
EOF


kubectl create -f ${WORKDIR}/csr.yaml

kubectl certificate approve vault.svc

kubectl get csr vault.svc -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out ${WORKDIR}/vault.crt

kubectl config view \
--raw \
--minify \
--flatten \
-o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
| base64 -d > ${WORKDIR}/vault.ca

### Create the Kubernetes namespace

kubectl create namespace $VAULT_K8S_NAMESPACE

### Create the TLS secret

kubectl create secret generic vault-ha-tls \
-n $VAULT_K8S_NAMESPACE \
--from-file=vault.key=${WORKDIR}/vault.key \
--from-file=vault.crt=${WORKDIR}/vault.crt \
--from-file=vault.ca=${WORKDIR}/vault.ca

### Deploy the Cluster

helm install vault hashicorp/vault --namespace vault -f values.yaml

kubectl -n $VAULT_K8S_NAMESPACE get pods

### Initialize vault-0 with one key share and one key threshold

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator init \
-key-shares=1 \
-key-threshold=1 \
-format=json > ${WORKDIR}/cluster-keys.json

### Unseal Vault running on the vault-0 pod.

VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" ${WORKDIR}/cluster-keys.json)

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

## Join vault-1 & vault-2 pods to the Raft cluster

### Join vault-1 

kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-1 -- /bin/sh

vault operator raft join -address=https://vault-1.vault-internal:8200 
-leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" 
-leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" 
-leader-client-key="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200

exit

kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY

### Join vault-1

kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-2 -- /bin/sh

vault operator raft join -address=https://vault-2.vault-internal:8200 
-leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" 
-leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" 
-leader-client-key="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200

exit

kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY

## Login to vault-0 with the root token

export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault login $CLUSTER_ROOT_TOKEN

## port forward the vault service

kubectl -n vault port-forward service/vault 8200:8200

## Destroy resources & namespace

kubectl delete all --all -n vault

kubectl delete namespace  vault
