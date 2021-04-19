# Quay Installation

Installation steps for Quay Registry, Quay Container Storage, and Quay Bridge.

## Prerequisites

Make sure OCS is installed.

1.  Install the OpenShift Container Storage operator through OperatorHub

2.  Create storage class

```
oc apply -f ./storage.yaml
```

## Install Quay

1.  Install the Quay operator through OperatorHub

2.  Create namespace for registry

```
oc new-project quay-registry
```

3.  Create Quay registry, this takes a few minutes

```
oc create -f ./quay.yaml

# this command will give you the Quay endpoint
oc describe QuayRegistry quay-registry -n quay-registry | yq e ".Status" - | grep 'Registry Endpoint' | sed 's/Registry Endpoint: //g'
```

4.  Open the Quay endpoint in a new url and create a new user and set a password, rest of steps assuming this quay endpoint is `https://quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com`

## Install Quay Container Security Operator

1.  Install the Quay Container Security Operator through the OperatorHub

## Install Quay Bridge Operator

### Installation

1.  Create organization in Quay, call it `openshift`

2.  Create New Token

    Go to `https://quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com/organization/openshift?tab=applications` and create a new application.  Give it all the permissions and agree to everything.  You will then get a single token.  Copy it.

3.  Create secret with token you copied

```
# assuming token is nNyomNGEfIHIBCesUzrYfIAkmAhwzOVh9C2Vvli4
oc create secret generic quay-integration -n openshift-operators --from-literal=token=nNyomNGEfIHIBCesUzrYfIAkmAhwzOVh9C2Vvli4
```

4.  Create Webhook

```
oc create -f ./quay-webhook.yaml
```

5.  Generate Cert

```
./webhook-create-signed-cert.sh --namespace openshift-operators --secret quay-bridge-operator-webhook-certs --service quay-bridge-operator
```

6.  Get generated Cert

```
oc get configmap -n kube-system \
   extension-apiserver-authentication \
   -o=jsonpath='{.data.client-ca-file}' | base64 | tr -d '\n'
```

7.  Apply Cert

Using the value you obtained above, replace `caBundle` from the `quay-mutating-webhook.yaml` file.  Then apply it.

```
# replace caBundle value
oc create -f ./quay-mutating-webhook.yaml
```

8.  Install Quay Bridge Operator through the OperatorHub.  Wait a few minutes for this to complete.

9.  Create namespace for watching.  In this case we will name it `hello`

```
oc new-project hello
```

10.  Modify Quay Hostname and Apply

```
# replace `quayHostname` in `quay-integration.yaml` with yours
oc apply -f ./quay-integration.yaml
```

### Fix Certs

1. Download certificate

   I generally use Chrome.  The filename will be something like `quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com@1617644668.cer`

2. Generate `crt` file from `cer` file

```
openssl x509 -inform DER -in ./quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com@1617644668.cer -out cert.crt
```

3. Add Cert to OCP as a configmap named `registry-cas`

```
oc create configmap registry-cas -n openshift-config --from-file=quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com=cert.crt
```

4. Patch OCP to include this new cert

```
oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}' --type=merge
```

5.  Verify the change was correct

```
oc edit image.config.openshift.io/cluster
```

6.  Add this cert to the CSO

```
oc create secret generic container-security-operator-extra-certs --from-file=cert.crt -n openshift-operators
```

7.  Restart the CSO Operator pod, changes won't mount inside the running pod

### Add Cert to Local Machine - Mac Only

To add the certificate to your local keychain means you'll be able to push the image directly into your Quay instance.

1.  Add cert to keychain

```
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ./cert.crt
```

2.  Restart docker

3.  Login to quay registry

```
docker login quay-registry-quay-quay-registry.apps.cluster-b6a3.b6a3.sandbox1438.opentlc.com
```
