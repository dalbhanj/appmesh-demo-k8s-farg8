# Demo Appmesh and mesh both K8s and Farg8 workloads

@ Inspired By:  
-- Github: https://github.com/PaulMaddox/aws-appmesh-helm  
-- Github: https://github.com/enghwa/cdkcolorteller   
-- Github: https://github.com/aws/aws-app-mesh-examples

## Deploy Fargate CDK components

```
# configure AWS region
$ export AWS_REGION=us-west-2
$ export AWS_DEFAULT_REGION=us-west-2

$ cd cdkcolorteller

# ensure we have node 8
$ nvm install 8.14.0
$ nvm alias default v8.14.0

# install aws cdk ver 0.28.0
$ npm i -g aws-cdk@0.28.0

# install node modules
$ npm install

# transpile ts to js
$ npm run build

# cdk bootstrap s3 bucket and deploy
$ cdk bootstrap
$ cdk deploy

# make sure you have docker installed and running or else you will get docker build error

# this will create all Fargate resources and output the ALB endpoint for Vue app

# Once cdk is deployed, edit App.vue, line 27 and assign the ALB DNS Name to "inputurl"
$ vi vueapp/src/App.vue

# Let CDK redeploy the VueJS container.
$ npm run build
$ cdk deploy
```

## Create EKS cluster in the same VPC as Fargate

```
# create a config file for our cluster (replace the region, vpc id, az's and subnet ID's with yours)

$ cd ..
$ cat << EOF > cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: appmesh-demo
  region: us-west-2
vpc:
  id: "vpc-01916cdb9af0b13c5"
  subnets:
    private:
      us-west-2a:
        id: "subnet-02df6460ee653107b"  
      us-west-2b:
        id: "subnet-09841b694c3192b5e"  
      us-west-2c:
        id: "subnet-043609911265d0577"                  
nodeGroups:
  - name: default
    instanceType: m5.large
    desiredCapacity: 2
    privateNetworking: true
    iam:
      withAddonPolicies:
        albIngress: true
        autoScaler: true
        appMesh: true
        xRay: true
EOF

$ eksctl create cluster -f cluster.yaml

# check if you can list the nodes in the cluster
$ kubectl get nodes
```

3) Setup a K8s service account for Helm (kubernetes package manager) and install Helm

cat <<EoF > helm-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF

$ kubectl apply -f helm-rbac.yaml

# install Helm into the K8s cluster
$ helm init --service-account=tiller
$ helm ls

4) Deploy helm chart to install k8s appmesh components (aws-appmesh-controller, aws-appmesh-grafana, aws-appmesh-inject, aws-appmesh-prometheus)

$ cd appmesh-demo-k8s-farg8
$ helm install -n aws-appmesh --namespace appmesh-system .

5) Create colorapp service
$ cd colorapp-appmesh-demo
$ helm install -n colorapp-appmesh-demo .


# Test in k8s
$ kubectl run -n appmesh-demo -it curler --image=tutum/curl /bin/bash
# curl colorgateway:9080/color
# for i in {1..10}; do curl colorgateway:9080/color; echo; done

# Edit weights

$ kubectl edit VirtualService colorteller.appmesh-demo -n appmesh-demo


6) Cleanup
helm del --purge aws-appmesh-demo
helm del --purge aws-appmesh
kubectl delete crds \
    meshes.appmesh.k8s.aws \
    virtualnodes.appmesh.k8s.aws \
    virtualservices.appmesh.k8s.aws
kubectl delete ns appmesh-demo appmesh-system



## TROUBLESHOOTING ###

# Error: release colorapp-appmesh-demo failed: object is being deleted: meshes.appmesh.k8s.aws "colormesh" already exists. Follow the steps to resolve
https://github.com/kubernetes/kubernetes/issues/60538

# patch custom resource
kubectl patch meshes/colormesh -p '{"metadata":{"finalizers":[]}}' --type=merge

# delete mesh
kubectl delete mesh/colormesh

## If it still is an issue
# patch crd
kubectl patch crd/meshes.appmesh.k8s.aws -p '{"metadata":{"finalizers":[]}}' --type=merge

# delete crd
kubectl delete crds/meshes.appmesh.k8s.aws
customresourcedefinition.apiextensions.k8s.io "meshes.appmesh.k8s.aws" deleted
