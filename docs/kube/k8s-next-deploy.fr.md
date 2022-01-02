---
title: Deployment
---


| Status | Date |
| --- | ---|
| <span className="dx-good">**APPROVED**</span>| [22nd of April 2021](https://pages.git.cwp.pnp-hcl.com/Team-Q/development-doc/architecture-community/2021-04-22) |

**All the stuff which is documented here, it is was only created under a deployment perspective.**

Operations, undeploy and version updates are not part of this architecture concept.

## Requirements

- Transparency of resources and the whole deployment
- Try to meet the Kube standards in terms of different flavours
- Ongoing support for new versions of Kube API's
- Single point of configuration truth

### Transparency of resources and the whole deployment
    
The complexity of our current deployment is to large. Customers have no real chance to determine how many resources will be consumed before they actually deploy it. The new deployment strategy need a clear summary of:

- min/max number of pods per service
- requested and limit of CPU&RAM
- scaling triggers
- required disk space
- resulting minimal footprint
- resulting maximum footprint

Also is a deployment via a operator extremely non-transparent. A customer has no changes to get information about:

- What is going via the deployment
- What will be deployed
- What will be changed automatically
- ...

### Try to meet the Kube standards in terms of different flavours.

The different kuberenetes flavours offer additional features to distinguish themselves in the market. While this is nice for individual application development, DX supports multiple vendors. Therefore it is very important that we are only following the kubernetes standard and not supporting some special functionality of other kubernetes flavour.

Example of a special flavour functionality:
Kubernetes is using Ingress to provide load balancing, SSL termination and name-based virtual hosting.
OpenShift additionally offers Routes which provides a bit more functionality than Ingress. Since OpenShift is kubernetes compatible Ingress is also available. https://www.openshift.com/blog/kubernetes-ingress-vs-openshift-route 

To minimize our maintenance and support cost, it is crucial that we are not using or relying on special functionalities outside the kube standard.

### Ongoing support for new versions of Kube API's.

Kubernetes and its internal API are rapidly changing. Therefore relying on the  `kubectl` cmd is not an ideal way. We have no option to ensure that all deployments are working correctly with newer versions of kube.
For that it would be ideal to have a framework as an abstraction layer which is our single interface to communicate with kube.

There are [multiple kube client libraries available](https://kubernetes.io/docs/reference/using-api/client-libraries/). Some of theme are officially-supported and other are maintained by an open source community.
We should only go with a officially-supported library.

Officially-supported language are:

- Golang
- Python
- Java
- dotnet
- Javascript/Typescript
- Haskel

To reduce the number of different languages we need to maintain within our development organization we should limit ourselves to one of these:

- Java
- Javascript/Typescript

### Single point of configuration truth

From an easy maintaining and supporting perspective it is really important to have a single point of configuration which will be used for both deployments ways Helm-based and DXCTL-based.

## Future design
### Met some proposals for a new deployment approach

For the new future design we have met the following proposals to align the future design with the requirements.

1. Using Helm charts as our single point of configuration
2. Supporting DXCTL as a alternative deployment way
3. DXCTL should be implemented new with a other language then golang
5. Don't use an operator
6. Automatically roll deployment - provide for each application/deployment a separated ConfigMap and provide a Global ConfigMap for generic configurations
7. Using a master consensus logic to executing changes on a multi pod deployment (like the execution of a ConfigTask on DX)

### Deployment designs
#### HELM-based deployment
OLD DIAGRAM

#### DXCTL-based deployment

OLD DIAGRAM

### Detail decisions informations

#### Single configuration point for HELM and DXCTL deployment
##### HELM Chart structure

```
- Charts.yaml
- values.yaml
- templates/service.yaml
- templates/deployments.yaml
- ...
```

The `values.yaml` file provided a way to collect all required values on a central place.
The properties form `values.yaml` can be used in each template.

##### Exports YAML files

With the following CMD it is possible to export all templates. During the export process all used `values.yaml` properties will be replaced with the right value.

```
helm template <CHART-NAME>
```

A example output of this CMD is a text based output and should be looks like that one:

```
---
# Source: mychart/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: RELEASE-NAME-mychart
  labels:
    helm.sh/chart: mychart-0.1.0
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
...
```

This output can be stored as an YAML file and can be apply to a cluster via the kubectl tool.

The DXCTL tool needs a new functionality to apply a bunch of YAML files to a kube cluster.

#### New implementation of the DXCTL

In our DX organization we have extremely less experience in GoLang. The most common skills are present in Java and Javascript/Typescript. Therefore it is from an maintaining and supporting perspective important to write the DXCTL tool in one of this language. Besides of that is the actual code quality not so good.

- Missing linting
- Missing tests
- Missing use of global constants
- A non-transparent proprietary mapping concept
- Bad code structure
- Contained unused code
- Code only copied from the operator


All in all it makes more sense to write the DXCTL tool new.

#### Don't use an operator for the deployment

What are the key features at the moment of our `DX CLOUD OPERATOR`?

- Scaling
- Triggering of some task (like a DX ConfigEngine task)
- Decided to use Routes (OpenShift) and the Ambassador for the rest
- Installation of our apps (RingAPI, CC, DAM)

All this task are also possible to do that without an operator.

The full status quo of the operator you can find [here](core-operator-status-quo).

**What are the main reason why we should don't use an operator for the deployment?**

The work of an operator is to non-transparent. Customers don't like it if they don't know what's happened during the deployment.


#### Automatically roll deployment - provide for each application/deployment a separated ConfigMap and provide a Global ConfigMap for all together

For a automatically roll deployment it would be good to have the configuration attributes are separated by deployments. Therefore it is possible to check the special ConfigMap if the values have changed and the special pods must be deployed again. 

Helm can helps us here with a checksum. The checksum of a special ConfigMap must be a part of the deployment metadata annotation.

https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments


#### Using a leader consensus logic to executing changes on a multi pod deployment (like the execution of a ConfigTask on DX)

The leader consensus logic we need to find which is the leader from the multi pods. Only of the leader it is possible to run a ConfigEngin task. This task will change the wp-profile which is used by all core pods.

Kubernetes self used [etcd](https://github.com/etcd-io/etcd) as a consensus system. [etcd](https://github.com/etcd-io/etcd) is using the [Raft](https://raft.github.io/) consensus algorithm.

Here is a really nice [article](https://blog.container-solutions.com/raft-explained-part-1-the-consenus-problem) and [demo](http://thesecretlivesofdata.com/raft/) of Raft.