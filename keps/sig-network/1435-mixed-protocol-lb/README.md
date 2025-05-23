# KEP-1435: different protocols in the same service definition with type=loadbalancer

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    - [Alibaba](#alibaba)
    - [AWS](#aws)
    - [Azure](#azure)
    - [DigitalOcean](#digitalocean)
    - [GCE](#gce)
    - [IBM Cloud](#ibm-cloud)
    - [OpenStack](#openstack)
    - [Oracle Cloud](#oracle-cloud)
    - [Tencent Cloud](#tencent-cloud)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Billing perspective](#billing-perspective)
    - [API change and upgrade/downgrade situations](#api-change-and-upgradedowngrade-situations)
- [Design Details](#design-details)
  - [Option Control Alternatives - considered alternatives](#option-control-alternatives---considered-alternatives)
    - [Annotation in the Service definition](#annotation-in-the-service-definition)
    - [Merging Services in CPI](#merging-services-in-cpi)
  - [The selected solution for the option control](#the-selected-solution-for-the-option-control)
  - [Kube-proxy](#kube-proxy)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha Graduation](#alpha-graduation)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
    - [Removing a Deprecated Flag](#removing-a-deprecated-flag)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Downgrade Strategy](#downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Summary

This feature enables the creation of a LoadBalancer Service that has different port definitions with different protocols. 


## Motivation

The ultimate goal of this feature is to support users that want to expose their applications via a single IP address but different L4 protocols with a cloud provided load-balancer. 
The following issue and PR shows considerable interest from users that would benefit from this feature:
https://github.com/kubernetes/kubernetes/issues/23880
https://github.com/kubernetes/kubernetes/pull/75831

The current restriction that rejects a Service creation request if it has two different protocol definitions was introduced because some cloud implementations may charge their load balancer users on a "per protocol" basis. That is, the current logic is to prevent negative surprises with the load balancer bills. The current implementation enforces a more explicit statement or consent form the end user for the usage of two different protocols on the same load balancer. For example GCE expects that the user creates two load balancer Service definitions for the two different protocols. 

But such workarounds or solutions do not exist in all cloud implementations. According to the feedback from the end users it would be more beneficial to remove this restriction from the Kubernetes code. This KEP is to investigate how the removal of that restriction would affect the billing of load balancer resources in the different clouds, i.e. whether it is safe or not to allow the usage of different protocol values in the same Service definition.

### Goals

The goals of this KEP are:

- To analyze the impact of this feature with regard to current implementations of cloud-provider load-balancers
- define how the activation of this feature could be made configurable if certain cloud-provider load-balancer implementations do not want to provide this feature

### Non-Goals

N/A

## Proposal

The first thing proposed here is to relax the API validation in Kubernetes that currently rejects Service definitions with different protocols if their type is LoadBalancer. Kubernetes would not reject Service definitions like this from that point:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mixed-protocol
spec:
  type: LoadBalancer
  ports:
    - name: dns-udp
      port: 53
      protocol: UDP
    - name: dns-tcp
      port: 53
      protocol: TCP
  selector:
    app: my-dns-server
  ```

Once that limit is removed those Service definitions will reach the Cloud Provider LB controller implementations. The logic of the particular Cloud Provider LB controller and of course the actual capabilities and architecture of the backing Cloud Load Balancer services determine how the actual exposure of the application really manifests. For this reason it is important to understand the capabilities of those backing services and to design this feature accordingly.

### User Stories

#### Story 1

As a Kubernetes cluster user I want to have the capability of exposing a DNS server for TCP and UDP based requests on the same Load Balancer IP address. See [RFC7766](https://tools.ietf.org/html/rfc7766).  In order to achieve this I want to define different `ports` with mixed protocols in my Service definition of `type: LoadBalancer`

#### Story 2

As as Kubernetes cluster user I want to have the capability of exposing my SIP Server for TCP and UDP based requests on the same Load Balacer IP address and port. This requirement comes from [RFC3261 Section 18.2.1](https://tools.ietf.org/html/rfc3261#section-18.2.1)

### Implementation Details/Notes/Constraints

#### Alibaba

The Alibaba Cloud Provider Interface Implementation (CPI) supports TCP, UDP and HTTPS in Service definitions and can configure the SLB listeners with the protocols defined in the Service.

The Alibaba SLB supports TCP, UDP and HTTPS listeners. A listener must be configured for each protocol, and then those listeners can be assigned to the same SLB instance. 

The number of listeners does not affect SLB pricing.
https://www.alibabacloud.com/help/doc-detail/74809.htm

A user can ask for an internal TCP/UDP Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"`. Internal SLBs are free.

Summary: Alibaba CPI and SLB seem to support mixed protocol Services, and the pricing is not based on the number of protocols per Service.

#### AWS

The AWS CPI supports TCP and UDP protocols in Service definitions when an AWS NLB is requested. However, the AWS CPI cannot set up an AWS NLB with TCP and UDP ports behind the same IP address for the same port number currently (for example, the DNS use case that wants to open port 53 with both UDP and TCP would not work right now). See https://github.com/kubernetes/kubernetes/pull/92109#discussion_r439730341

For AWS ELB only TCP is accepted by the CPI. 

The AWS CPI looks for annotations on the Service to determine whether TCP, TLS or HTTP(S) listener should be created in the AWS ELB for a configured Service port.

AWS Classic LB supports TCP,TLS, and HTTP(S) protocols behind the same IP address. 

AWS Network LB supports TCP/TLS and UDP protocols behind the same IP address. 

The usage of TCP+HTTP or UDP+HTTP on the same LB instace behind the same IP address is not possible in AWS.

From a pricing perspective the AWS NLB and the CLB have the following models:
https://aws.amazon.com/elasticloadbalancing/pricing/
Both are usage based rather than based on the number of protocols, however NLBs have separate pricing unit quotas for TCP, UDP and TLS.

A user can ask for an internal Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0`. So far the author could not find any difference in the usage and pricing of those when compared to the external LBs - except the pre-requisite of a private subnet on which the LB can be deployed.

Summary: AWS CPI supports mixed protocols (TCP and UDP) on AWS NLBs, but the support of the same port number with different protocols still requires some work in the CPI . Once this feature is implemented in the K8s API the users can request for such mixed-protocol NLBs via their Service definitions, and the users will be charged according to the rules defined in the pricing rules.

#### Azure

Azure CPI LB documentation: https://github.com/kubernetes-sigs/cloud-provider-azure/blob/master/docs/services/README.md

The Azure CPI already supports the usage of both UDP and TCP protocols in the same Service definition. It is achieved with the CPI specific annotation `service.beta.kubernetes.io/azure-load-balancer-mixed-protocols`. If this key has value of `true` in the Service definition, the Azure CPI adds the other protocol value (UDP or TCP) to its internal Service representation. Consequently, it also manages twice the amount of load balancer rules for the specific frontend. 

Only TCP and UDP are supported in the current mixed protocol configuration.

The Azure Load Balancer supports only TCP and UDP as protocols. HTTP support would require the usage of the Azure Application Gateway. I.e. HTTP and L4 protocols cannot be used on the same LB instance/IP address

Basic Azure Load Balancers are free.

Pricing of the Standard Azure Load Balancer is based on load-balancing rules and outbound rules.  There is a flat price up to 5 rules, on top of which every new forwarding rule has additional cost. 
https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview#pricing

A user can ask for an internal Azure Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/azure-load-balancer-internal: true`. There is not any limitation on the usage of mixed service protocols with internal LBs in the Azure CPI.

Summary: Azure already has mixed protocol support for TCP and UDP, and it already affects the bills of the users. The implementation of this feature may require some work in Azure CPI.

#### DigitalOcean

The DigitalOcean CPI does not support mixed protocols in the Service definition, it accepts TCP only for load balancer Services. It is possible to ask for HTTP(S) and HTTP2 ports with Service annotations.

Summary: The DO CPI is the current bottleneck with its TCP-only limitation. As long as it is there the implementation of this feature will have no effect on the DO bills.

#### GCE

The GCE CPI supports only TCP and UDP protocols in Services.

GCE/GKE creates Network Load Balancers based on the K8s Services with type=LoadBalancer. In GCE there are "forwarding rules" that define how the incoming traffic shall be forwarded to the compute instances. A single forwarding rule can support either TCP or UDP but not both. In order to have both TCP and UDP forwarding rules we have to create separate forwarding rule instances for those. Two or more forwarding rules can share the same external IP if
- the network load balancer type is External
- the external IP address is not ephemeral but static

There is a workaround in GCE, please see this comment from the original issue: https://github.com/kubernetes/kubernetes/issues/23880#issuecomment-269054735 If the external IP is static the user can create two Service instances, one for UDP and another for TCP and the user has to specify that static external IP address in those Service definitions in `loadBalancerIP`.

HTTP protocol support: there is a different LB type in GCE for HTTP traffic: HTTP(S) LB. I.e. just like in the case of AWS it is not possible to have e.g. TCP+HTTP or UDP+HTTP behind the same LB/IP address.

Forwarding rule pricing is per rule: there is a flat price up to 5 forwarding rule instances, on top of which every new forwarding rule has additional cost. 

[GCP forwarding_rules_charges](https://cloud.google.com/compute/network-pricing#forwarding_rules_charges) suggest that the same K8s Service definition would result in the creation of 2 forwarding rules in GCP. This has the same fixed price up to 5 forwarding rule instances, and each additional rule results in extra cost.

A user can ask for an internal TCP/UDP Load Balancer via a K8s Service definition that also has the annotation `cloud.google.com/load-balancer-type: "Internal"`. Forwarding rules are part of the GCE Internal TCP/UDP Load Balancer architecture, too, and the user can define different forwarding rules with different protocols for the same IP address.

Summary: The implementation of this feature can affect the bills of GCP users. However the following perspectives are also observed:
- if a user wants to have UDP and TCP ports behind the same NLB two Services with must be defined, one for TCP and one for UDP. As the pricing is based on the number of forwarding rules this setup also means the same pricing as with the single Service instance.
- if a user is happy with 2 NLB (Service) instances for TCP and UDP still the user has two more forwarding rules to be billed - i.e. it has the same effect on pricing as if those TCP and UDP endpoints were behind the same NLB instance

That is, if a user wants to use different protocols on the same LB that can be achieved with 2 Service definitions with the current GCP services now. It is not the number of LBs or Service definitions that matters. This phenomenon is already there with the current practice, and the enabling of mixed protocols will not change it to the worse.

#### IBM Cloud

The IBM Cloud creates VPC Load Balancer in a VPC based cluster and NLB in a classic cloud based cluster. 

The IBM Cloud CPI implementation for the classic cluster supports TCP and UDP protocol values in K8s Services, and it supports different protocol values in a Service. 

The IBM Cloud CPI implementation for the VPC clusters supports only TCP. 

The VPC Load Balancer supports TCP and HTTP, and it is possible to create TCP and HTTP listeners for the same LB instance. UDP is not supported. 

The VPC LB pricing is time and data amount based, i.e. the number of protocols on the same LB instance does not affect it.

The NLB supports TCP and UDP on the same NLB instance. The usage of NLB does not have pricing effects, it is part of the IKS basic package.

Summary: once this feature is implemented in the K8s API Server the IBM Cloud VPC LB can still use only TCP ports from a Service definition. NLB can use TCP and UDP ports from a single Service definition.

#### OpenStack

The OpenStack CPI supports TCP, UDP, HTTP(S) in Service definitions and can configure the Octavia listeners with the protocols defined in the Service.
OpenStack Octavia supports TCP, UDP and HTTP(S) on listeners, an own listener must be configured for each protocol, and different listeners can be used on the same LB instance.

There was a bug in Octavia versions <5.0.0: it was not possible to use the same port number (5e.g. 53) with different protocols (e.g. TCP and UDP) on the same LB instance. It has been fixed in 5.0.0, which is available since the "T" release of OpenStack.

Summary: the OpenStack based clouds that use Octavia v2 as their LBaaS seems to support this feature once implemented. It is true that the "T" release is the newest one, so upgrade may take a while. On the other hand OpenStack documentation mentions, that a newer Octavia version can be used with previous releases of other OpenStack projects, i.e. it can be the case that the upgrade effort is on Octavia side in an OpenStack cloud. 
Pricing is up to the pricing model of the OpenStack providers.

#### Oracle Cloud

The Oracle CPI does not support UDP in the K8s Service definition. It supports only TCP and HTTP. It supports mixed TCP and HTTP ports in the same Service definition.

The Oracle Cloud Load Balancer supports TCP, HTTP(S) protocols. 

The pricing is based on time and capacity: https://www.oracle.com/cloud/networking/load-balancing-pricing.html I.e. the amount of protocols on a sinlge LB instance does not affect the pricing.

Summary: Oracle CPI and LB seem to support mixed protocol Services, and the pricing is not based on the number of protocols per Service.

#### Tencent Cloud

The Tencent Cloud CPI supports TCP, UDP and HTTP in Service definitions. It maps "HTTP" protocol value in a Service definition to "TCP" before creating a listener on the CLB.
The Tencent Cloud CPI can manage both of their LB solutions: "Classic CLB" and "Cloud Load Balancer" (previously known as "Application Load Balancer"). 
The Tencent Cloud CLB supports TCP, UDP and HTTP(S) listeners, an own listener must be configured for each protocol. 
The number of listeners does not affect CLB pricing. CLB pricing is time (day) based and not tied to the number of listeners.
https://intl.cloud.tencent.com/document/product/214/8848

A user can ask for an internal Load Balancer via a K8s Service definition that  has the annotation `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxxxxx`. Internal CLBs are free.

Summary: Tencent Cloud CPI and LBs seem to support mixed protocol Services, and the pricing is not based on the number of protocols per Service.

### Risks and Mitigations

#### Billing perspective

The goal of the current restriction on the K8s API was to prevent an unexpected extra charging for Load Balancers that were created based on Services with mixed protocol definitions.
If the current limitation is removed without any option control we introduce the same risk. Let's see which clouds are exposed:
- Alibaba: the pricing here is not protocol or forwarding rule or listener based. No risk.
- AWS: there is no immediate impact on the pricing side as the AWS CPI limits the scope of protocols to TCP only.
- Azure: Azure pricing is indeed based on load-balancing rules, but this cloud already supports mixed protocols via annotations. There is another risk for Azure, though: if the current restriction is removed from the K8s API, the Azure CPI must be prepared to handle Services with mixed protocols.
- GCE: here the risk is valid once the user exceeds the threshold of 5 forwarding rules.
- IBM Cloud: no risk
- OpenStack: here the risk is that there is almost no chance to analyze all the public OpenStack cloud providers with regard to their pricing policies
- Oracle: no risk
- Tencent Cloud: no risk

#### API change and upgrade/downgrade situations

We relax here a validation rule, which is considered as an API-breaking act by the [K8s API change guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md) Even if the change is implemented behind a feature flag the following actions can cause problems:
- the user creates a Service with mixed protocols and then
  - turns off the feature flag; or
  - executes K8s version rollback to the N-1 version where this feature is not available at all

When investigating the possible issues with such a change we must consider [the supported version skew among components](https://kubernetes.io/docs/setup/release/version-skew-policy/), which are the K8s API server and the cloud controller managers (CPI implementations) in our case.

First of all, feature gate based (a.k.a conditional) field validation must be implemented as defined in the [API change guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#alpha-field-in-existing-api-version) One can see a good example for a case when an existing field got a new optional value [here](https://github.com/kubernetes/kubernetes/issues/72651). This practice ensures that upgrade and rollback between _future releases_ is safe. Also it enables the further management of existing API objects that were created before turning off the feature flag. Though it does not save us when a K8s API server version rollback is executed to a release in which this feature is not implemented at all.

Our feature does not introduce new values or new fields. It enables the usage of an existing value in existing fields, but with a different logic. I.e. if someone creates a Service with mixed protocol setup and then rollbacks the API server to a version that does not implement this feature the clients will still get the Service with mixed protocols when they read that via the rollback'd API. If the client (CPI implementation) has been rollbacked, too, then the client may receive such a Service setup that it does not support.

- Alibaba: no risk. The current CPI and LB already supports the mixed protocols in the same Service definition. If this feature is enabled in an API server and then the API server rollback is executed the CPI can still handle the Services with mixed protocol sets.
- AWS: no risk. The current CPI and LB already supports the mixed protocols in the same Service definition. The situation is the same as with the Alibaba CPI.
- Azure: no risk. The current CPI and LB already supports the mixed protocols in the same Service definition. The situation is the same as with the Alibaba CPI.
- GCE: currently the GCE CPI assumes that a Service definition contains a single protocol value, as it assumes the apiserver already rejects Services with mixed protocols during validation. While the Service Controller really did so a while ago, it does not do this anymore. It means a risk. However, Google team members stated in the enhancement issue that we could proceed.
- DigitalOcean: no risk. The current CPI accepts Services with TCP protocol only, i.e. after a K8s upgrade a user still cannot use this feature. Consequently, a rollback in the K8s version does not introduce any issues.
- IBM Cloud VPC: no risk. The same situation like in the case of AWS.
- IBM Cloud Classic: no risk. The CPI and NLB already supports TCP and UDP in the same Service definition. The same situation like in the case of Alibaba.
- OpenStack: no risk. The CPI and NLB already supports TCP and UDP in the same Service definition. The same situation like in the case of Alibaba.
- Oracle: no risk. The CPI and LB already supports mixed protocols. The same situation like in the case of Alibaba.
- Tencent Cloud: no risk. The CPI and LB already supports mixed protocols. The same situation like in the case of Alibaba.

As stated above we must implement a feature gate based phased introduction for this feature because of its effects. See the `The selected solution for the option control` part for details in this document below.

## Design Details

The implementation of the basic logic is ready in this PR:
https://github.com/kubernetes/kubernetes/pull/75831

The current implementation has a feature gate to control its activation status.

### Option Control Alternatives - considered alternatives

#### Annotation in the Service definition

In this alternative we would have a new annotation in the `kubernetes.io` annotation namespace, as it was planned in the original PR. Example: `kubernetes.io/mixedprotocol`. If this annotation is applied on a Service definition the Service would be accepted by the K8s API.

Pro: 
- a kind of "implied conduct" from the user's side. The user explicitly defines with the usage of the annotation that the usage of multiple protocols on the same LoadBalancer is accepted
- Immediate feedback from the K8s API if the user configures a Service with mixed protocol set but without this annotation
Con:
- Additional configuration task for the user
- Must be executed for each and every Service definitions that are to define different protocols for the same LB

#### Merging Services in CPI

This one is not really a classic option control mechanism. The idea comes from the current practice implemented in MetalLB: https://metallb.universe.tf/usage/#ip-address-sharing
The same works in GCE as a workaround.

I.e. if a cloud provider wants to support this feature the CPI must have a logic to apply the Service definitions with a common key value (for example loadBalancerIP) on the same LoadBalancer instance. If the CPI does not implement this support it will work as it does currently.
Pro:
- the cloud provider can decide when to support this feature, and until that it works as currently for this kind of config
- the restriction on mixed protocols can remain in the K8s API - i.e. there is no K8s API change, and the corresponding risks are mitigated
Con:
- the users must maintain two Service instances
- Atomic update can be a problem - the key must be such a Service attribute that cannot be patched

### The selected solution for the option control

In the first release: 
 - a feature flag shall control whether new loadbalancer Services with mixed protcol configuration can be created or not
 - we must add a note to the documentation that if such Service is created then it may break things after a rollback - it depends on the cloud provider implementation
 - the update of such Services shall be possible even if the feature flag is OFF. This is to prepare for the next release when the feature flag is removed from the create path, too, and after a rollback to the first release the update of existing Service objects must be possible
 - the CPI implementations shall be prepared to deal with Services with mixed protocol configurations. Either via supporting such Service definitions, or clearly indicating to the users that the Service could not be processed as specified. As we can see from our analysis some CPIs support other protocols than TCP and UDP in the Service definitions, while others support only TCP and UDP. That is the term "mixed protocol support" does not always mean that all possible protocol values are supported by a CPI. For this reason a nicely behaving CPI shall 
 - indicate clearly to the user what ports with what protocols have been opened on the LB
 - preferably not create any Cloud LB resources if the Service definition contains unsupported protocols.
 
 In order to provide a way for the CPI to indicate port statuses to the user we would add the following new `portStatus` list of port statuses to `Service.status.loadBalancer.ingress`, so the CPI can indicate the status of the LB. Also we add a `conditions` list of conditions to the `Service.status.loadBalancer`, with the first official condition `LoadBalancerMixedProtocolNotSupported` defined.

```json
"io.k8s.api.core.v1.ServiceCondition": {
  "description": "ServiceCondition contains details for the current condition of this Service.",
  "properties": {
    "type": {
      "description": "Type is the type of the condition. Known conditions are \"LoadBalancerMixedProtocolNotSupported\".\n\nThe  \"LoadBalancerMixedProtocolNotSupported\" condition with \"True\" status means that the cloud provider implementation could not create the requested load-balancer with the specified set of ports because that set contains different protocol values for the ports, and such a configuration is not supported either by the cloud provider or by the load balancer or both.",
      "type": "string"
    },
    "status": {
      "description": "Status is the status of the condition. Can be True, False, Unknown.",
      "type": "string"
    },
    "message": {
      "description": "Human-readable message indicating details about last transition.",
      "type": "string"
    },
    "reason": {
      "description": "Unique, one-word, CamelCase reason for the condition's last transition.",
      "type": "string"
    }
  },
  "required": [
    "type",
    "status"
  ],
  "type": "object"
},
"io.k8s.api.core.v1.PortStatus": {
  "description": "PortStatus contains details for the current status of this port.",
  "properties": {
    "port": {
      "description": "Port number",
      "format": "int32",
      "type": "integer"
    },
    "protocol": {
      "description": "The protocol for this port",
      "type": "string"
    },
    "ready": {
      "description": "Specifies whether the port was configured for the load-balancer.",
      "type": "boolean"
    },
    "message": {
      "description": "A human readable message indicating details about why the port is in this condition.",
      "type": "string"
    }
  },
  "type": "object"
},
"io.k8s.api.core.v1.LoadBalancerIngress": {
  "description": "LoadBalancerIngress represents the status of a load-balancer ingress point: traffic intended for the servicshould be sent to an ingress point.",
  "properties": {
    "hostname": {
      "description": "Hostname is set for load-balancer ingress points that are DNS based (typically AWS load-balancers)",
      "type": "string"
    },
    "ip": {
      "description": "IP is set for load-balancer ingress points that are IP based (typically GCE or OpenStack load-balancers)",
      "type": "string"
    },
    "portStatuses": {
      "description": "The list has one entry per port in the manifest.",
      "items": {
        "$ref": "#/definitions/io.k8s.api.core.v1.PortStatus"
      },
      "type": "array"
    },
    "message": {
      "description": "A human readable message indicating details about the condition of the load-balancer.",
      "type": "string"
    }
  },
  "type": "object"
},
"io.k8s.api.core.v1.LoadBalancerStatus": {
  "description": "LoadBalancerStatus represents the status of a load-balancer.",
  "properties": {
    "ingress": {
      "description": "Ingress is a list containing ingress points for the load-balancer. Traffic intended for the servi  should be sent to these ingress points.",
      "items": {
        "$ref": "#/definitions/io.k8s.api.core.v1.LoadBalancerIngress"
      },
      "type": "array"
    }
  },
  "type": "object"
},
"io.k8s.api.core.v1.ServiceStatus": {
  "description": "ServiceStatus represents the current status of a service.",
  "properties": {
    "loadBalancer": {
      "$ref": "#/definitions/io.k8s.api.core.v1.LoadBalancerStatus",
      "description": "LoadBalancer contains the current status of the load-balancer, if one is present."
    },
    "conditions": {
      "description": "Current service state",
      "items": {
        "$ref": "#/definitions/io.k8s.api.core.v1.ServiceCondition"
      },
      "type": "array",
      "x-kubernetes-patch-merge-key": "type",
      "x-kubernetes-patch-strategy": "merge"
    }
  },
  "type": "object"
},
```

A CPI shall also set an Event in case it cannot create a Cloud LB instance that could fulfill the Service specification.

In the second release: 
- the feature flag shall be set to ON by default (promoted to beta). Most probably we want to keep the feature flag so cloud providers can decide whether they enable it or not in their managed K8s services depending their CPI implementations.

In the long term:
- the feature flag is removed and the feature becomes generic without option control

### Kube-proxy

Kube-proxy will not block traffic based on the port status information from `Service.status.loadBalancer.ingress` that a load balancer controller sets. Because multi-protocol is a feature, clusterIP and NodePort traffic will work independent of the load balancer. Because some load balancers use NodePort and some use VIP_like LB traffic, there may exist packets to LBIP:LBPORT/wrong-protocol. Traffic will not be blocked based on variation in this implementation detail.


### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

Proper unit tests are in place for the affected code parts.

See:
https://github.com/kubernetes/kubernetes/blob/867b5cc31b376c9f5d04cf9278112368b0337104/pkg/registry/core/service/strategy_test.go#L315
https://github.com/kubernetes/kubernetes/blob/ea0764452222146c47ec826977f49d7001b0ea8c/pkg/apis/core/validation/conditional_validation_test.go#L29

##### Integration tests

There shall be integration tests that verify that the API server accepts Services with mixed protocols.


##### e2e tests

The change described in this KEP affects provider specific implementations (CPIs) when considering the e2e aspect. 
Within the k8s e2e test set we shall check whether the kube-proxy works well with a Service that has type=LoadBalancer and has different protocols defined.

### Graduation Criteria

From end user's perspective the graduation criteria are feedback/bug correction and testing based.

From CPI implementation perspective the feature can be graduated to beta, as the cloud providers with managed K8s products can still decide whether they activate it for their managed clusters or not, depending on the status of their CPI implementation.

Graduating to GA means, that the feature flag checking is removed from the code. It means, that all CPI implementations must be ready to deal with Services with mixed protocol configuration - either rejecting such Services properly or managing the cloud load balancers according to the Service definition.

#### Alpha Graduation

- Feature is implemented and controller with a feature flag. The feature flag is disabled by default.

#### Alpha -> Beta Graduation

- All of the major clouds support this or indicate non-support properly

#### Beta -> GA Graduation

- All major cloud providers support this or indicate non-support properly

#### Removing a Deprecated Flag

TBD

### Upgrade / Downgrade Strategy

#### Downgrade Strategy

If a user creates loadbalancer Services with mixed protocols and then the user downgrades to an API Server version that does not support this feature at all the only operation the user can execute on those Services is the delete operation.

If the downgrade also affects the version of the CPI implementation (i.e. when a K8s API server rollback implicitly executes a CPI implementation rollback, too) then the new CPI implementation version may not handle the existing cloud load balancers on a consistent way. It is recommended that the user deletes such Services before the rollback is started to such K8s API server/CPI implementation that does not support this feature.

The same stands for the case when the user wants to move the existing Service definitions to another K8s cluster: the user shall check whether the target K8s cluster supports this feature or not and modify the Service descriptors accordingly.

### Version Skew Strategy

Version skew is possible among the following components in this case: K8s API server, CPI implementation, cloud load balancer

Once this feature is implemented in the API server there is a chance that the CPI implementation has to deal with load balancer Services with mixed protocol configuration anytime, even if the API server is downgraded later. The CPI implementation shall be prepared for this, i.e. the CPI implementation cannot expect anymore that the API Server (or any other component, like the Service Controller) filters out such Service definitions. If the CPI implementation wants to have such filtering it has to implement that on its own. In this case the CPI implementation shall be upgraded before the feature is enabled on the API server. In case of a rollback of the API Server such Services can still exist in the cluster, so the CPI implementation should not be downgraded to a version that does not implement that filtering. This is the reason why the CPI implementations shall be updated (if necessary) to be able to deal with such Service definitions already in the first release of this feature.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: MixedProtocolLBService
    - Components depending on the feature gate: Kubernetes API Server

* **Does enabling the feature change any default behavior?**

  When the feature is enabled the Services with mixed protocols are not rejected anymore by the Kubernetes API server, and it is up to the CPI to handle those.
  Please see the analysis at `API change and upgrade/downgrade situations`

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**

  Yes. 

* **What happens if we reenable the feature if it was previously rolled back?**

  Nothing serious, as this feature removes a restriction in the API Server. I.e. this direction does not introduce problems.

* **Are there any tests for feature enablement/disablement?**

  TBD

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**

  Enabling this feature gate moves the responsibility for handling mixed protocols from the Kubernetes API server to the specific CPI. See the provider-specific details at [API change and upgrade/downgrade situations](#api-change-and-upgradedowngrade-situations).

* **What specific metrics should inform a rollback?**

  As all providers either already have adjusted their error messages or intend to, enabling this feature gate may lead to a CPI error. If load balancer traffic only works correctly for one protocol and not for the other, that is another reason to roll back. In such cases, remediation would be achieved by disabling the feature gate.

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**

This feature uses an existing value in existing fields, but with a different logic. If we create a Service with mixed protocol and then we roll back the API server to a version that does not implement this feature, the clients will still get the Service with mixed protocols when using the rolled-back API. If the client (CPI implementation) has been roll backed as well, then the client may encounter a Service setup that it does not support. All clouds either support this or adjusted their errors reported, per the enhancement issue.


### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**

After checking to see if a Service of `type:LoadBalancer` that uses two different protocols on the same port was created and validated by the API server, the operator can then check the specifics of the [API change and upgrade/downgrade situations](#api-change-and-upgradedowngrade-situations) for their specific cloud provider; this covers object persistence and availability.

The e2e tests shall check that
- a multi-protocol Service triggers the creation of a multi-protocol cloud load balancer 
Optionally, if the CPI supports that:
- the CPI sets the new Conditions and or Port Status in the Load Balancer Service after creating the cloud load balancer

* **What are the SLIs (Service Level Indicators) an operator can use to determine 
the health of the service?**
  - [x] Metrics
    - Metric name: kube-state-metrics/kube_service_status_load_balancer_ingress
      - Reference: https://github.com/kubernetes/kube-state-metrics/blob/12402a564cbf4557763079ab8e6e995d9afb4db9/docs/service-metrics.md   
      - Components exposing the metric: Service via the kube-state-metrics add-on
      - If the LoadBalancer Service has neither IP address nor Hostname the Service is not healthy  

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**

Before this KEP, expectations of performance for this specific feature were consistent across providers. In this KEP we loosen validation on the Services API, which allows the creation of Services with two protocols where before the API server only allowed one protocol per Service. Depending on CPI and cloud provider API implementation, setup of additional ports or listeners with different protocols may take more time than was previously required for single-protocol Services.


* **Are there any missing metrics that would be useful to have to improve observability 
of this feature?**

Once this feature becomes stable kuber-state-metrics could be enhanced to support the new Service.Status.Condition "LoadBalancerPortsError". If that condition is true for the Service the load balancer ports of the Service could not be allocated successfully.

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  CPIs shall be prepared to handle Service definitions with mixed protocols. Please see the analysis above.


### Scalability

* **Will enabling / using this feature result in any new API calls?**

  If a CPI supports the management of the new Conditions and PortStatus in the LoadBalancer Service the management of those fields will mean additional traffic on the API

* **Will enabling / using this feature result in introducing new API types?**

 No

* **Will enabling / using this feature result in any new calls to the cloud 
provider?**

  If the cloud provider requires more calls to add ports/listeners with different protocols to a load balancer then this feature introduces additional calls

* **Will enabling / using this feature result in increasing size or count of 
the existing API objects?**

  Yes. As detailed above, the Status of the Service is planned to be extended with new Conditions and PortStatus

* **Will enabling / using this feature result in increasing time taken by any 
operations covered by [existing SLIs/SLOs]?**

  The setup of more ports/listeners with different protocols may take more time, depending on how the CPI and the cloud provider API is implemented

* **Will enabling / using this feature result in non-negligible increase of 
resource usage (CPU, RAM, disk, IO, ...) in any components?**

  Not expected.

### Troubleshooting

* **How does this feature react if the API server and/or etcd is unavailable?**

The CPI sets the Conditions and/or PortStatus on the Service.Status object. If the API service is not available, the CPI cannot update the status. If the CPI updates the status and then later the API server becomes unavailable, the Status is stored with the Service object and will be available the next time the API server starts.

It's possible for the CPI to set the new Conditions and/or PortStatus in the Load Balancer Service after creating the cloud load balancer, but this feature will not be triggerable without API server response.

* **What are other known failure modes?**

Cloud providers will need to provide their intended responses; in most cases they intend to initially indicate non-support, then add support later.

* **What steps should be taken if SLOs are not being met to determine the problem?**

Enabling this feature gate moves the responsibility for handling mixed protocols from the Kubernetes API server to the specific CPI. Depending on CPI and cloud provider API implementation, setup of additional ports or listeners with different protocols may take more time than was previously required for single-protocol Services. Diagnosis for any performance impacts will be specific to the out-of-tree cloud providers.

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- the `Proposal` section being merged, signaling agreement on a proposed design: 14th July 2020
- Move from alpha to beta after agreement from all listed providers, April 2022

## Drawbacks

 TBD

## Alternatives

 No known alternatives, beside those analysed above.

## Infrastructure Needed (Optional)

 The Cloud Load Balancer shall support the setup of different protocols on the same IP address in order to utilize the benefits of this feature.
 The CPI implementation shall either indicate problems with the Service in the new status fields, or the CPI shall be able to setup the cloud load balancer with the different protocols.
