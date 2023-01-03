# keda-autoscaler-example
Simple example of using Keda auto scaler to scale deployment based on cpu utilization and using IAM ROLE as authentication


## Install keda 
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

## Custom values to create and pass IAM ROLE ARN
```
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "<IAM ROLE ARN>"
  create: true
  name: <service-account-name>
```

With the custom values upgrade the helm chart
`helm upgrade keda kedacore/keda --namespace keda -f custom-values.yaml`

## Create IAM Policies followed by Roles
- Create Policies with required permission to be given 
- Edit the Role to add the Trust relationships
- Add the serviceAccount associated with the keda Scaler as following 

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::548053982756:oidc-provider/oidc.us-west-2.eks.amazonaws.com/XXXXX"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.us-west-2.eks.amazonaws.com/XXXXXX:sub": "system:serviceaccount:<NAMESPACE>:<SERVICE-ACCOUNT-NAME>"
                }
            }
        }
    ]
}
```


## Create a simple nginx deployment
Use the file `nginx-deploy.yaml` to deploy a simple nginx deployment
If you want to use your own, make sure to have resource limits being set else the HPA wont work. 

## Create TriggerAuthentication
Each TriggerAuthentication is defined in one namespace and can only be used by a ScaledObject in that same namespace. For cases where you want to share a single set of credentials between scalers in many namespaces

Here, we will simply create TruggerAuthentication that utilize AWS IAM Role so it has to be of provider type `aws-kiam`. Best from security prospective as well instead of using AWS access-key and secrets. 

filename: triggerAuthentication.yaml

```
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-iam
  namespace: keda-test
spec:
  podIdentity:
    provider: aws-kiam
```

## Finally Create a ScaledObject that will scale the Deployment based on CPU utilization:

filename: scaledObject.yaml
```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp-cpu-utilization
  namespace: keda-test
spec:
  maxReplicaCount: 3
  minReplicaCount: 1
  scaleTargetRef:
    # Name of the deployment to be Scaled 
    name: nginx-deployment
  triggers:
  - metadata:
      type: Utilization
      value: "80"
    type: cpu
    authenticationRef:
      # Name of the authentication type to be implemented
      name: keda-trigger-auth-aws-iam
```

