# cp4mcm-runtime-management
Repository hosting IBM Cloud Pak for Multicloud Management Manage Runtime metadata

## 1.0 Overview
Runtime custom resources provide a way to define a customizable and central management pane  for workloads that run across VMs and Clusters in IBM's Cloud Pak for Multicloud Management (CP4MCM). For more information on the usage, prerequisites, and features provided through CP4MCM's Manage Runtime feature, please see the knowledge center documentation <>. For more information on how to create additional views beyond the ones included in this repository, see below.


## 2.0 Fields/Structure
Define the below to **enable your runtime to show in the overview cards menu and show top level row data**, applications (for cluster based runtimes), and default links.

To see end to end examples, please see the existing Custom Resources provided in the  [manage_runtime_crs](manage_runtime_crs/) folder.


| Field | description | Required | Form |
| ----- | ----------- | ---------| ------ |
| metadata.name | Name of your runtime CR | Y | string |
| metadata.namespace | Namespace | Y | string |
| spec.displayName | Name of your runtime to be used in display cards on landing page | Y | string |
| spec.description | Description to be used in the display cards on landing page  | Y | string |
| spec.search.X where X is cluster-based and/or vm-based | More below | Y. One or both of spec.search.vm-based and spec.search.cluster-based must be defined. More below | Object. More below |
| spec.search.X.label where X is vm-based or cluster-based | Kubernetes label=value that can be used to determine Kubernetes resources that are of your runtime type | Y | string. Example: "app.management.ibm.com/ibm-mq=true" |
| spec.search.X.kind where X is vm-based or cluster-based | Kubernetes type that represent the top level resource of your runtime. Follows MCM Search object types (you can test/explore types using MCM Search UI)| Y | string. singular. Example: "statefulset" |
| spec.search.cluster-based.links | List of search criteria to describe external routes that you would like exposed on the Manage Runtime Interface for cluster-based resources. See 2.1 below | N | Array of JSON objects. See 2.1 below |


### 2.1 Links
The ```spec.search.cluster-based.links``` section can be used to describe external routes that you would like to expose on the Manage Runtimes User Interface. Here, you can specify 0-N routes that will appear in the overflow menu (accessible via the vertical ellipsis icon) of a runtime instance, if a respective route is found within the bounds of the search criteria for a specific instance.

An example of what this looks like in the console is depicted below.

![IBM MQ Console Link (smaller view)](img/IBM%20MQ%20Console%20Link%20(smaller%20view).png)

![IBM MQ Console Link (larger view)](img/IBM%20MQ%20Console%20Link%20(larger%20view).png)

Please that **we current only support showing dynamic links for Openshift Routes**. We may extend this to support showing generic Kubernetes ingress paths in the future. If you are running on Openshift, and your workload exposes a service, but not a route, you may expose it via ```oc expose service yourServiceName -n yourServiceNamespace```

The general form for a **single** link search criteria is as follows:

| Field | description | Required | Form |
| ----- | ----------- | ---------| ------ |
| labels | Kubernetes labels that can be used to uniquely identify your target route. Numerous labels will be treated as AND | Y | Array of strings|
| sameNamespace | Whether the target route must exist in the same namespace as the parent workload | Y | boolean |
| containsName | String that can be used to further tune your search parameter. See 2.1.1 for examples | N | string |
| matchWorkloadPrefix | Signifies whether the target route has the same name prefix as the parent workload (typically happens when artifacts are from the same release). This can be used to further tune your search criteria. See 2.1.1 for examples  | Y | boolean |
| security | Whether the target route uses http or https | Y | String. "http" or "https"|
| displayName| Display Name for target route, as seen in Manage Runtimes UI | Y | string |


#### 2.1.1 Link Examples
##### IBM MQ : One to One w/ name based filtering
Here, we will demonstrate how to define a link search criteria in the case that (1) a single route maps to a single runtime instance, (2) multiple releases may exist in the same namespace, and (3) multiple routes are deployed per instance, and we would like to employ name based filtering.

Consider the case of IBM MQ. For MQ, we would like to expose the User Interface Console.

Describing the runtime instances (as defined by the ```spec.search.cluster-based.label``` and ```spec.search.cluster-based.kind```):

```
[root@bronze-cluster-inf ~]# kubectl get statefulset --selector=app=ibm-mq
NAME                 READY   AGE
mq-dev-app-ibm-mq    1/1     13d
mq-prod-app-ibm-mq   1/1     12d
```

And describing the routes using the ```app=ibm-mq``` label:
```
[root@bronze-cluster-inf ~]# kubectl get route --selector=app=ibm-mq
NAME                     HOST/PORT                                                                  PATH   SERVICES             PORT   TERMINATION   WILDCARD
mq-dev-app-ibm-mq-qm     mq-dev-app-ibm-mq-qm-message-board.apps.bronze-cluster.os.fyre.ibm.com            mq-dev-app-ibm-mq    1414   passthrough   None
mq-dev-app-ibm-mq-web    mq-dev-app-ibm-mq-web-message-board.apps.bronze-cluster.os.fyre.ibm.com           mq-dev-app-ibm-mq    9443   passthrough   None
mq-prod-app-ibm-mq-qm    mq-prod-app-ibm-mq-qm-message-board.apps.bronze-cluster.os.fyre.ibm.com           mq-prod-app-ibm-mq   1414   passthrough   None
mq-prod-app-ibm-mq-web   mq-prod-app-ibm-mq-web-message-board.apps.bronze-cluster.os.fyre.ibm.com          mq-prod-app-ibm-mq   9443   passthrough   None
```

For the mq-dev-app-ibm-mq runtime instance I would like to expose the mq-dev-app-ibm-mq-web route. For the mq-prod-app-ibm-mq runtime instance I would like to expose the mq-prod-app-ibm-mq route.


Here, because the route must exist in the same namespace as the workload, we will set ```sameNamespace: true```. Because each release is associated with its own route (ie one to one) we will set ```matchWorkloadPrefix:true```. Finally, to filter out the routes not associated with the web console, we will set ```containsName: "ibm-mq-web"```. Note, if the target route had employed a more explicit label set, such as including the label, ```mq.managment.web=true```, it could have sufficed to supply an additional label instead of using ```containsName```

The full links definition is as follows:
```
links:
- labels:
    - app=ibm-mq
  sameNamespace: true
  matchWorkloadPrefix: true
  containsName: ibm-mq-web
  security: https
  displayName: "MQ Console"
```



##### IBM ACE: One to Many
Here, we will demonstrate how to define a link search criteria in the case that (1) a single route maps to multiple runtime instances and (2) the labels held by the route are explicit enough to not require further name based filtering.

Consider the case of IBM ACE. For IBM ACE Server, multiple instances may be deployed in a single project, but the UI (which is deployed through a separate release) serves all server instances in the project.

Describing the runtime instances (as defined by the ```spec.search.cluster-based.label``` and ```spec.search.cluster-based.kind```):
```
[root@bronze-cluster-inf stable]# kubectl get deployment --selector=app.kubernetes.io/name=ibm-ace-server-dev
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
ace-install-ibm-ace-server-dev    3/3     3            3           3h58m
ace-install2-ibm-ace-server-dev   3/3     3            3           90s
```

And describing the routes using the ```app.kubernetes.io/name=ibm-ace-dashboard-dev``` label (note this is a different label that was used to define the parent runtime in ```spec.search.cluster-based.label``` because the dashboard component was deployed through a separate release and contains different labels):

```
[root@bronze-cluster-inf stable]# kubectl get route --selector=app.kubernetes.io/name=ibm-ace-dashboard-dev
NAME          HOST/PORT                                                 PATH   SERVICES                         PORT        TERMINATION   WILDCARD
ace-dash-ui   ace-dash-ui-ace-dev.apps.bronze-cluster.os.fyre.ibm.com          ace-dash-ibm-ace-dashboard-dev   controlui   reencrypt     None
```

For both the ace-install-ibm-ace-server-dev and ace-install2-ibm-ace-server-dev instances, I would like to expose the ace-dash-ui route


Here, because a route suffices to serve all of the server instance in the project, and the label is restrictive enough to identify the route, it will suffice to use the labels and sameNamespace parameters to identify it (as seen below). If multiple routes existed that held the target route label, we could use the containsName property to further filter.

The full links definition is as follows:
```
links:
- labels:
    - app.kubernetes.io/name=ibm-ace-dashboard-dev
  sameNamespace: true
  matchWorkloadPrefix: false
  security: https
  displayName: "ACE Dash UI"
```
