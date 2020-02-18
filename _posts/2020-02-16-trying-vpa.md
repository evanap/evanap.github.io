---
layout: post
title: "The Only Recommendation We Need is the Ones Provided by the Vertical Pod Autoscaler"
date: 2020-02-16
---

Right-sizing a deployment of an app on a Kubernetes clulster is a tricky thing to do. Setting requests and limits too high, you might end up with unschedullable pods due to resource scarcity. Setting requets and limits too low and you might end up with OOMKilled pods. Luckily, there's an answer (kind of) for that. It's the **Vertical Pod Autoscaler** or **VPA** for short.

VPA is a custom resource, a part of autoscaling components of Kubernetes. It can scale resource and limits of the containers in pods, based on their usage over time. Very neat.

## Installation

Installation of the VPA is pretty straightforward. You just need to clone the repo [here](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) and run the provisioning script:

```
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/veretical-pod-autoscaler
./hack/vpa-up.sh
```

*Note: you need cluster-admin role to provision the VPA system components*

## Usage

If you've used the Horizontal Pod Autoscaler (HPA) in the past, then using the VPA is not much different. You just need to create a VPA object that points to your target deployment. Below is an example of a VPA definition:

```yaml
---
apiVersion: "autoscaling.k8s.io/v1beta2"
kind: VerticalPodAutoscaler
metadata:
  name: your-app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: your-app
  updatePolicy:
    updateMode: "Off"
```

In the example I define the **updateMode** of the VPA. Currently, there are three available update modes of the VPA:
- Auto: Makes updates directly to your pods
- Recreate: Evict old pods and create new ones with updated definition
- Off: Not applying any recommendations

For me, setting the update mode is safest way since there's no modification taking place until we apply the recommendation ourselves.

If the VPA is successfully created, after about 5 minutes you will start getting resource recommendation of the targeted deployment based on its usage history. You can grab the recommendation by describing the VPA:


```
kubectl describe vpa your-app-vpa
```

And it will look like this:
```
Name:         your-app-vpa
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"autoscaling.k8s.io/v1beta2","kind":"VerticalPodAutoscaler","metadata":{"annotations":{},"name":"your-app-vpa","namespace":"d...
API Version:  autoscaling.k8s.io/v1beta2
Kind:         VerticalPodAutoscaler
Metadata:
  Creation Timestamp:  2020-02-16T14:04:28Z
  Generation:          29
  Resource Version:    8472916
  Self Link:           /apis/autoscaling.k8s.io/v1beta2/namespaces/default/verticalpodautoscalers/your-app-vpa
  UID:                 086f993f-09b7-461d-ba50-7f2cedf2e95b
Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         your-app-vpa
  Update Policy:
    Update Mode:  Off
Status:
  Conditions:
    Last Transition Time:  2020-02-16T14:06:38Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  your-app
      Lower Bound:
        Cpu:     556m
        Memory:  131072k
      Target:
        Cpu:     587m
        Memory:  131072k
      Uncapped Target:
        Cpu:     587m
        Memory:  131072k
      Upper Bound:
        Cpu:           17347m
        Memory:        318166666
      Container Name:  istio-proxy
      Lower Bound:
        Cpu:     12m
        Memory:  173661513
      Target:
        Cpu:     12m
        Memory:  183046954
      Uncapped Target:
        Cpu:     12m
        Memory:  183046954
      Upper Bound:
        Cpu:     304m
        Memory:  5064299060
Events:          <none>
```

Now that we've got the recommendation provided by the VPA, we can set our deployment resource to not get below the **Lower Bound** and maybe set it around the recommended **Target** values. 


## Problems

On my time experimenting with the VPA I faced one weird issue. One VPA in the cluster reports that it has "No pods match this VPA object." Given that, it's still giving correct recommendations for the containers of the target deployment.

```
Name:         server-app-vpa
Namespace:    core
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"autoscaling.k8s.io/v1beta2","kind":"VerticalPodAutoscaler","metadata":{"annotations":{},"name":"server-app-vpa","namespace...
API Version:  autoscaling.k8s.io/v1beta2
Kind:         VerticalPodAutoscaler
Metadata:
  Creation Timestamp:  2020-02-16T14:21:19Z
  Generation:          11
  Resource Version:    8472225
  Self Link:           /apis/autoscaling.k8s.io/v1beta2/namespaces/core/verticalpodautoscalers/server-app-vpa
  UID:                 c0e41569-2216-41cb-abe8-febf4017460b
Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         server-app
  Update Policy:
    Update Mode:  Off
Status:
  Conditions:
    Last Transition Time:  2020-02-16T14:21:38Z
    Message:               No pods match this VPA object
    Reason:                NoPodsMatched
    Status:                True
    Type:                  NoPodsMatched
    Last Transition Time:  2020-02-16T14:21:38Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  server-app
      Lower Bound:
        Cpu:     70m
        Memory:  1190888880
      Target:
        Cpu:     163m
        Memory:  1312092764
      Uncapped Target:
        Cpu:     163m
        Memory:  1312092764
      Upper Bound:
        Cpu:           10283m
        Memory:        66464285183
      Container Name:  istio-proxy
      Lower Bound:
        Cpu:     20m
        Memory:  184882685
      Target:
        Cpu:     109m
        Memory:  203699302
      Uncapped Target:
        Cpu:     109m
        Memory:  203699302
      Upper Bound:
        Cpu:     6382m
        Memory:  10318423263
Events:          <none>
```

## Conclusion

The VPA is a very useful feature and I think it can bring a whole lot of optimizations to our Kubernetes clusters. Though, for now it still need some improvements and to get rid of the bugs like the one I mentioned before. Maybe if it matured a bit more we will see use cases of it in a production environment. For now, I'm keeping it for staging environments and maybe combine it with traffic shadowing to get a grasp of recommendation without using it in production.
