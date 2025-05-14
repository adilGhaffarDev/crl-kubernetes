## Kubeconfig revokation with crl

Link to patched code for k8s v1.33.0: https://github.com/Nordix/kubernetes/commits/kubeconfig-revoke-1.33/adil

- Create kind cluster
```
kind create cluster
```

- Load the patched apiserver image:
```
kind load image-archive patched-1-33-image/kube-apiserver.tar --name kind --nodes kind-control-plane
```
- Set the image:
```
docker exec kind-control-plane sed -i 's|image: .*|image: registry.k8s.io/kube-apiserver-amd64:v1.13.0-alpha.0.59663_0871373ead991e-dirty|' /etc/kubernetes/manifests/kube-apiserver.yaml
```

- Create new admin kubeconfig:
```
# Create new certs

# Create admin key
openssl genpkey -algorithm RSA -out admin.key

# Create admin csr
openssl req -new -key admin.key -out admin.csr -subj "/CN=admin/O=system:masters"

# copy ca key and crt from kind-control-plane to current folder
docker cp kind-control-plane:/etc/kubernetes/pki/ca.key .
docker cp kind-control-plane:/etc/kubernetes/pki/ca.crt .

# Create admin crt
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 365 -sha256


# Create new kubeconfig

# Change path to ca.crt according to where it was copied in previous step
kubectl config set-cluster kind-kind --server=https://127.0.0.1:33841 --certificate-authority=/home/metal3ci/go/src/k8s.io/kubernetes/certs/ca.crt --kubeconfig=new-admin-kubeconfig.yaml

# Change path to admin.key and admin.crt according to where they were created in previous step
kubectl config set-credentials new-admin --client-certificate=/home/metal3ci/go/src/k8s.io/kubernetes/certs/admin.crt --client-key=/home/metal3ci/go/src/k8s.io/kubernetes/certs/admin.key --kubeconfig=new-admin-kubeconfig.yaml

kubectl config set-context new-admin@kind-kind --cluster=kind-kind --user=new-admin --kubeconfig=new-admin-kubeconfig.yaml

kubectl config use-context new-admin@kind-kind --kubeconfig=new-admin-kubeconfig.yaml

# Test it
kubectl --kubeconfig=new-admin-kubeconfig.yaml get pods -A
```

- Extract client.crt from kubeconfig

```
#!/bin/bash

USER=$(kubectl config view --minify -o jsonpath='{.users[0].name}')
CERT=$(kubectl config view --raw -o jsonpath="{.users[?(@.name==\"$USER\")].user['client-certificate-data']}")
KEY=$(kubectl config view --raw -o jsonpath="{.users[?(@.name==\"$USER\")].user['client-key-data']}")

echo "$CERT" | base64 -d > client.crt
echo "$KEY"  | base64 -d > client.key
```

- Create crl
```
sudo openssl ca -config /etc/ssl/openssl.cnf -gencrl -keyfile ca.key -cert ca.crt -out ca.crl

# Add client.crt to ca.crl, client.crt is the crt you want to revoke, you can get it from kubeconfig.
sudo openssl ca -revoke client.crt -keyfile ca.key -cert ca.crt
sudo openssl ca -gencrl -keyfile ca.key -cert ca.crt -out ca.crl

# Copy ca.crl to a folder in control plane node that is accessible by kube-apiserver
docker cp /path/to/ca.crl kind-control-plane:/etc/kubernetes/pki/ca.crl
# set ca.crl it in the /etc/kubernetes/manifests/kube-apiserver.yaml on control-plane
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add --certificate-revocation-list as below
#spec:
#  containers:
#  - command:
#    - kube-apiserver
#    - --advertise-address=...
#    - --allow-privileged=true
#    - --certificate-revocation-list=/etc/kubernetes/pki/ca.crl
```

Kubeapi-server will restart and once that is done, you will no longer be able to use the kubeconfig that has client that is revoked.
You should still be able to access the server with new-admin-kubeconfig.yaml that we created.
```
kubectl --kubeconfig=new-admin-kubeconfig.yaml get pods -A
```

Once crl file is added, there is no need to restart kubeapiserver whenever crl file is updated. kubeapiserver needs to be restarted only when crl file is added for the first time.

Some things to consider:

One control plane
- There is no crl file from start but flag is set:
    - It will fail, will not be able to access kubeapiserver(probably should not fail and notify user that file is missing, or have validation hook that does not allow user to set the variable to location where file is not there)

- There is empty crl file from the start but flag is set
    - Original patch this was failing, this is fixed now, in logs user will see log that crl file is empty.

- There was crl file in start but later deleted
    - Currently it is failing. Logically if file is deleted user should remove the flag too. Something that needs to be discussed how we should handle this.
- There was crl file in start but later emptied
    - All kubeconfigs will be able to access the apiserver again.


HA Cluster needs to be investigated.