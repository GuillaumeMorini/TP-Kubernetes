# Kubernetes Networking TP

Welcome to the Kubernetes networking TP! In this session you'll learn about networking in Kubernetes, using a single Kubernetes cluster.

## Contents

# Kubernetes Networking 

Networking in Kubernetes has advanced networking features - containers in pods can reach each other by DNS name, and Kubernetes can load-balance incoming requests across pods.

Kubernetes also has one useful networking feature - networking policies. Policies let you restrict traffic between containers in a way which is transparent to the application inside the container. In this section you'll learn how to use networking policies.

## Controlling Network access

Our Lab uses [Tigera's](https://www.tigera.io) [Project Calico](https://www.projectcalico.org/) for networking in Kubernetes. Calico implements Kubernete's [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) resources, allowing you to configure `ingress` and `egress` rules for pod traffic:

- `ingress` policies define which network sources are allowed to send traffic into a pod. These are useful for restricting service access to specific consumers;
- `egress` policies define which network targets a pod is allowed to
  send traffic to. These are useful for isolating services so they can
  only connect to specific components, and for blocking traffic
  leaving the cluster to external Internet addresses.
  
## The Demo App

The demo for this part of the workshop is a distributed application with multiple components. Version 1 of the app has a legacy backend service, which is fronted by a facade. There's a client application which talks to the facade to use backend services:

![version 1 application diagram](img/kube/dockercon-network-workshop-0.png)

Version 2 of the app introduces a payment service. The service isn't directly accessible - only via a queue. Backend services can publish request messages to the queue, and the payments service can process them and publish response messages to the queue:

![version 2 application diagram](img/kube/dockercon-network-workshop-1.png)

> The app has been badly configured though, so every service is trying to use every other service. We'll use network policies enforced by Calico to make sure traffic flows where it should.

## Deploy the Demo App

First, clone this repo and navigate to the `kubernetes` directory on the master node:

```
git clone https://github.com/GuillaumeMorini/TP-Kubernetes
cd ./TP-Kubernetes/kubernetes
```

Now deploy version 1 of the application:

```
kubectl apply -f ./manifests/
```

Check out the services in the Kubernetes `management-ui` namespace. This will allow you to visualize the traffic flows occurring between the micro-services in the application.  To do so, please follow these instructions.

Identify the node port used and open a new web browser tab/window with localhost and the node port behind (like `http://localhost:30002`). This should show the real-time application data flows.

Be sure to check that you forced your browser to use http and not https, as the app is based on HTTP.

If it all worked, you should now see something that looks like the image below

![](img/kube/calico-stars-no-policy.jpg)

This isn't good. The client is communicating directly with the backend, as well as with the facade, and the facade and backend are communicating with the client. We can fix that with ingress policies.

## Ingress Policies

To start with you'll write and deploy a blanket ingress policy which restricts all traffic to all pods. When this is enforced, pods will only be able to communicate if they have an ingress policy which specifies explicit access, overriding the default policy.

Deny all ingress for app namespaces by writing and deploying [default-deny.yaml](kubernetes/policies/ingress/default-deny.yaml), which sets up a network policy in each namespace (except management-ui) with these rules:

- for any pod in the namespace
- restrict ingress traffic to an empty list - i.e. allow no ingress traffic

```
kubectl apply -f ./policies/ingress/default-deny.yaml
```

> Refresh your management UI and after a few seconds you'll see a blank screen. The UI component can't even access the application pods to see where traffic is flowing!

The first thing is to allow ingress traffic from the management UI, so you can see what's happening. Write the file [allow-management-ui.yaml](kubernetes/policies/ingress/allow-management-ui.yaml) that sets up a policy in each namespace (except management-ui) with these rules:

- for any pod in the namespace
- allow ingress traffic from any pod in a namespace which matches the label `role=management-ui`

Deploy the policy to set up the UI:

```
kubectl apply -f ./policies/ingress/allow-management-ui.yaml
```

> Note that the YAML file applies the same network policy to all the application namespaces.

Now in the management UI you can see all the app components, but they're still isolated so there's no traffic between them:

![](img/kube/calico-stars-ui-policy.jpg)

We'll set up the correct access by applying new ingress rules, specifying where we want traffic to go. Write[backend-policy.yaml](kubernetes/policies/ingress/backend-policy.yaml) with rules to allow the facade to access the backend:

- in the namespace `backend`
- for any pods matching the label `app=backend`

```
kubectl apply -f ./policies/ingress/backend-policy.yaml
```

Refresh the management UI and you'll see traffic is no flowing from the facade to the backend, but the client is still isolated:

![](img/kube/calico-stars-backend-policy.jpg)

Next write [facade-policy.yaml](kubernetes/policies/ingress/facade-policy.yaml) that allows the client to access the facade:

```
kubectl apply -f ./policies/ingress/facade-policy.yaml
```

In the management UI you'll see that traffic is now flowing where we want it to go, and we are enforcing the correct data flow at the orchestration layer:

![](img/kube/calico-stars-frontend-policy.jpg)

> It's important to realise that these are the same pods you deployed at the start, and they are still *trying* to send traffic everywhere. The application hasn't been fixed, but the traffic is under control with ingress policies.

## Egress Policies

Version 2 of the app deploys the payment service running in the `pci` namespace, and the message queue running in the `infrastructure` namespace.

Deploy new version of the management UI and you'll see those components in the stack:

```
kubectl apply -f ./management-ui-v2.yaml
```

The queue and payments service are isolated because of the default ingress policies, blocking all non-whitelisted traffic:

![](img/kube/calico-queue-blocked.jpg)

We want some extra security around the payments service, because it needs PCI compliance. Write a default egress policy which denies any non-whitelisted traffic.

[pci-policy.yaml](kubernetes/policies/egress/pci-policy.yaml) enforces these rules:

- for any pods in the `pci` namespace
- block all `egress` traffic to any destination
- except on port `53`, which is explicitly permitted.

Apply that policy and you'll see the management UI doesn't change, because the payments service is already isolated by ingress rules:

```
kubectl apply -f ./policies/egress/pci-policy.yaml
```

> You typically need to allow egress traffic on port 53 for DNS - without that, pods cannot resolve DNS names to enforce other policies.

Now you can add an ingress policy which explicitly allows traffic to the queue. Write [queue-policy.yaml](kubernetes/policies/ingress/queue-policy.yaml) which does this:

- in the namespace `infrastructure`
- for pods matching the label `app=queue`
- allow ingress on TCP port 5000
- from any pod in a namespace matching the label `requires=queue`

Both the `pci` and `backend` namespaces specify that label, so when you deploy the policy it will allow apps to publish messages to the queue:

```
kubectl apply -f ./policies/ingress/queue-policy.yaml
```
This works for the `backend` and `facade` services, which can now publish messages. The `payments` service still can't send traffic - the ingress rule for the target destination allows it, but the egress rule from the source blocks it:

![](img/kube/calico-payments-blocked.jpg)

The final step is to allow egress from the `payments` service to the queue. Write [payments-policy.yaml](kubernetes/policies/egress/payments-policy.yaml) which allows this:

- in the `pci` namespace
- for pods matching the label `app=payments`
- allow egress on any port
- to any pod in a namespace matching the label `role=infrastructure `

Deploy the final policy to complete configuration of the app:

```
kubectl apply -f ./policies/egress/payments-policy.yaml
```

Now all the traffic is flowing as required, controlled by network policies:

![](img/kube/calico-fully-configured.jpg)

The default ingress rules prevent the application services receiving traffic from any new services without being explicitly allowed.

The default egress rule prevents any app which requires PCI compliance from sending traffic to any destination except infrastructure pods.

## Clean Up

Delete all the app namspaces:

```
kubectl delete ns frontend
kubectl delete ns backend
kubectl delete ns pci
kubectl delete ns infrastructure
kubectl delete ns management-ui
```

## Thank you

Do not forget to send me one report per group of student with details of the created files, and explanations of your choices.
Please send the report through email or Teams private message.

