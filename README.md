
<h1 align="center"><b>Automated Creation of Route53 Records with ExternalDNS</b></h1>

This is the description of how to implement automated creation of Route53 recods with ExternalDNS on AWS account from EKS cluster in another AWS account.

&nbsp;

## Project Description
ExternalDNS synchronizes exposed Kubernetes Services and Ingresses with DNS providers. It allows you to control DNS records dynamically via Kubernetes resources in a DNS provider-agnostic way. It retrieves a list of resources (Services, Ingresses, etc.) from the Kubernetes API to determine a desired list of DNS records. 
In this particular project we have an EKS cluster created in one AWS account and Route53 hosted zone in another AWS account. The goal is to automaticaly create DNS records in Route 53 when ingresses created. DNS record must be created only if special annotation present. 

#### Overall structure looks like this:
There is a ExternalDNS deployment that monitors the creation of ingresses. When ingress created it looks for annotatins. When <external-dns.alpha.kubernetes.io/include: "true"> annotated, it creates the DNS record in Route53. The host name will be the "spec.host" in ingress object. 

### The steps to install ExternalDNS:
Since we have two different AWS accounts, ExternalDNS needs to assume role in a different account. To do that we need to use chained AssumeRole operations. Create role in each account and configure ExternalDNS to assume a role in Route53 account.

#### Here is the steps we need to take: 
1. Create an IAM Policy and a role in Route53 account with permissions to create and list the records.
2. Create permissions to modify DNS zone for a EKS cluster.
3. Deploy ExternalDNS.
4. Test.

### 1. Route 53.
1. Create this policy to add permissions to change records in Route53.
   This is the policy:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

2. Create a role and attach that created policy.
3. Then add a Trust Relationship to the role we just created in Route53. You might want to put random name instead of             <ROLE_ARN_FROM_EKS_ACCOUNT> if you haven't created a role in EKS cluster account yet. Once you create a role in EKS cluster (in the following  steps), change <ROLE_ARN_FROM_EKS_ACCOUNT> to the actual name of the role from EKS cluster.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "<ROLE_ARN_FROM_EKS_ACCOUNT>"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

### 2. EKS cluster.

Before you continue, you need to make sure the EKS cluster is configured with an OIDC provider and add support for IRSA (IAM roles for Service Accounts) <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>. In this examples the Service Account name will be "external-dns".
1. Create a policy to assume a role from Route53:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "<ROLE_ARN_FROM_ROUTE53_ACCOUNT>"
        }
    ]
}
```
2. Create role and attach above policy.
3. Add a Trust Relashionship to to that role. Use your <ACCOUNT_NUMBER>, <OIDC_PROVIDER> and <NAMESPACE> values:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_NUMBER>:oidc-provider/<OIDC_PROVIDER>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "<OIDC_PROVIDER>:aud": "sts.amazonaws.com",
                    "<OIDC_PROVIDER>:sub": "system:serviceaccount:<NAMESPACE>:external-dns"
                }
            }
        }
    ]
}
```

### 3. Deploy ExternalDNS.
Before applying the ExternalDNS manifest file, we need to edit some arguments:
--source=service # Comment out this line, since we don't want to auto-creation of the DNS record on services (it only create records for type=LoadBalancer)
--aws-assume-role=ROUTE53_ROLE_ARN # This will let ExternalDNS assume role in Route53.
--domain-filter=example.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones.
--policy=upsert-only # Comment out this line to make sure records deleted when ingress resource deleted.
--annotation-filter=external-dns.alpha.kubernetes.io/include in (true) # This will ensure record only created when this annotation is present and set to "true", i.e. external-dns.alpha.kubernetes.io/include: "true"

Here is an yaml file:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods","nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: exdns # change to desired namespace: externaldns, kube-addons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: exdns
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.13.2
          args:
            #- --source=service
            - --aws-assume-role=<ROLE_ARN_FROM_ROUTE53_ACCOUNT>
            - --source=ingress
            - --domain-filter=exchangeweb.net # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            #- --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=external-dns
            - --annotation-filter=external-dns.alpha.kubernetes.io/include in (true)
          env:
            - name: AWS_DEFAULT_REGION
              value: us-east-1 
```

### 4. Test.
Create a sample nginx deployment with service and ingress resource to test the ExternalDNS (add your own values to ingress: <YOUR_INGRESS_CLASS> and <YOUR_HOSTNAME>):

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP 
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: exdns
  annotations:
    external-dns.alpha.kubernetes.io/include: "true"
spec:
  ingressClassName: <YOUR_INGRESS_CLASS>
  rules:
    - host: <YOUR_HOSTNAME>
      http:
        paths:
          - backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
```

### 5. Issues.

One issue I had is with Nginix-ingress Controller maintained by Nginx. Nginx-ingress has a different arguments for ExternalDNS. One of the arguments is needed to publish the endpoint of the ingress controller, so ExternalDNS can automatically use it. The endpoint is AWS load balancer. This is from the documentation for ExternalDNS:
"When you create the nginx-ingress-controller Deployment, you need to provide the --publish-service option to the /nginx-ingress-controller executable under args. Once this is deployed new Ingress resources will get the ELB's FQDN and ExternalDNS will use the same when creating records in Route 53."
I tested with args Nginx-ingress has on their official website and it didn't work. In the documentation of ExternalDNS they use ingress-nginx args. That's why I decided to create another one, which is Ingress-nginx maintained by Kubernetes. It also created separate service for it with the LB. I haven't tried to use same service for a different controller, not sure if its a good idea, need to research more. I also created separate IngressClass (ingress-nginx) for it, so I can choose which controller to use in ingress resource.

### Resource links.

ExternalDNS: 
https://github.com/kubernetes-sigs/external-dns

Setting up ExternalDNS for Services on AWS: 
https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-roles-for-service-accounts

Frequently asked questions: 
https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md

