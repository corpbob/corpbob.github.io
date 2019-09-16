---
layout: post
title: "How to Demo Serverless using Knative"
author: "Bobby Corpus"
categories: journal
tags: [documentation,sample]
image: 2019-08-21-introduction-to-knative/serverless3.png
---

# Assumptions

- You have OKD 3.11 up and running
- Clone the repo for this demo:

[https://github.com/corpbob/knativedemo.git](https://github.com/corpbob/knativedemo.git)

- This demo is based on the knativetutorial:

[https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial-basics/0.7.x/index.html](https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial-basics/0.7.x/index.html)

# Enable Admission Webhooks

- Execute the following command:

```
sed -i -e 's/"admissionConfig":{"pluginConfig":null}/"admissionConfig": {\
    "pluginConfig": {\
        "ValidatingAdmissionWebhook": {\
            "configuration": {\
                "apiVersion": "v1",\
                "kind": "DefaultAdmissionConfig",\
                "disable": false\
            }\
        },\
        "MutatingAdmissionWebhook": {\
            "configuration": {\
                "apiVersion": "v1",\
                "kind": "DefaultAdmissionConfig",\
                "disable": false\
            }\
        }\
    }\
}/' /etc/origin/master/master-config.yaml
```

- Restart the services

```
master-restart api
master-restart controllers
systemctl restart origin-node
```

- Install Istio

```
curl -LO https://github.com/knative/serving/releases/download/v0.5.0/istio-crds.yaml
curl -LO https://github.com/knative/serving/releases/download/v0.5.0/istio.yaml

sed -i 's/LoadBalancer/NodePort/g' istio.yaml
oc apply -f istio-crds.yaml
oc apply -f istio.yaml
```

- Install Knative Serving

```
curl -LO https://github.com/knative/serving/releases/download/v0.6.0/serving.yaml
oc apply -f serving.yaml
```

- Install Knative Eventing

```
curl -LO https://github.com/knative/eventing/releases/download/v0.6.0/eventing.yaml
curl -LO https://github.com/knative/eventing/releases/download/v0.6.0/in-memory-channel.yaml
curl -LO https://github.com/knative/eventing/releases/download/v0.6.0/release.yaml

oc apply -f eventing.yaml
oc apply -f in-memory-channel.yaml
oc apply -f release.yaml
```

- Demo Proper

```
oc new-project knativedemo
oc adm policy add-scc-to-user privileged -z default 
oc adm policy add-scc-to-user anyuid -z default

```

- Deploy the event-greeter

```
oc apply -f 01_service.yaml
```

- Call the service to see if it works

```
bash 02-call.sh
Hi  greeter => '9861675f8845' : 1
```

- Wait for 1 minute for the pod to scale down to zero. Once it scales down to zero, call the script again

```
bash 02-call.sh
```

Observe that the service scaled up to 1. This is what serverless means: The ability to scale down to zero when it's not being used.

- The next demo will show you how you can create different versions of your serverless application.

```
oc apply -f 03-configuration-rev1.yaml
oc apply -f 04-route.yaml
```

- Call the service

```
bash 05-call.sh 
Hi  greeter => '9861675f8845' : 1
```

- Get the current revisions of the service

```
bash 06-revisions.sh 
```

It should give you something like:

```
NAME            AGE
greeter-fd9xp   3m

```

- Create another version of the service

```
oc apply -f 07-configuration-rev2.yaml
```

You should get another deployment. Verify using

```
oc get deployments
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
greeter-fd9xp-deployment   0         0         0            0           5m
greeter-pfcvw-deployment   1         1         1            1           30s
```

- Call the service to get response from the new version

```
bash 08-call.sh 
OKD  greeter => '9861675f8845' : 1
```

- In a separate terminal, ssh to your machine and call the service in a loop:

```
[root@bcorpus7 knativedemo]# bash 09-call-loop.sh 
OKD  greeter => '9861675f8845' : 2
OKD  greeter => '9861675f8845' : 3
OKD  greeter => '9861675f8845' : 4
OKD  greeter => '9861675f8845' : 5
OKD  greeter => '9861675f8845' : 6
```

- Get the list of revisions we have of the new service:

```
[root@bcorpus7 knativedemo]# bash 10-revisions.sh 
NAME            AGE
greeter-fd9xp   9m
greeter-pfcvw   4m
```

- Now that we have 2 revisions, we can now set that all traffic go to the old revision. Edit the file 11-route-all-rev1.yaml and change the value of revision name to the name of the revision 1:

```
cat 11-route-all-rev1.yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: greeter
spec:
  traffic:
    - revisionName: greeter-fd9xp
      percent: 100
```

- Apply the configuration to OpenShift:

```
oc apply -f 11-route-all-rev1.yaml
```

You will notice that in the other terminal that calls the service in a loop, the output has changed.

```
OKD  greeter => '9861675f8845' : 73
OKD  greeter => '9861675f8845' : 74
OKD  greeter => '9861675f8845' : 75
Hi  greeter => '9861675f8845' : 1
Hi  greeter => '9861675f8845' : 2
Hi  greeter => '9861675f8845' : 3
Hi  greeter => '9861675f8845' : 4

```

- Now edit the file 12-route-all-rev2.yaml and change the value of revisionName to the name of the newest revision:

```
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: greeter
spec:
  traffic:
    - revisionName: greeter-pfcvw
      percent: 100
```

- Apply the configuration and watch the log of the loop. Notice that the loop output will now change to version 2:

```
oc apply -f 12-route-all-rev2.yaml

Hi  greeter => '9861675f8845' : 152
Hi  greeter => '9861675f8845' : 153
OKD  greeter => '9861675f8845' : 1
OKD  greeter => '9861675f8845' : 2
OKD  greeter => '9861675f8845' : 3
OKD  greeter => '9861675f8845' : 4
```

- Edit the file 13-route-split.yaml and specify the name of version 1 that will get 75% split and version 2 that will get 25% split of the traffic.

```
cat 13-route-split.yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: greeter
spec:
  traffic:
    - revisionName: greeter-fd9xp
      percent: 75
    - revisionName: greeter-pfcvw
      percent: 25
```

- Observe the difference in the output of the loop:

```
Hi  greeter => '9861675f8845' : 1
Hi  greeter => '9861675f8845' : 2
Hi  greeter => '9861675f8845' : 3
Hi  greeter => '9861675f8845' : 4
Hi  greeter => '9861675f8845' : 5
Hi  greeter => '9861675f8845' : 6
Hi  greeter => '9861675f8845' : 7
Hi  greeter => '9861675f8845' : 8
OKD  greeter => '9861675f8845' : 99
Hi  greeter => '9861675f8845' : 9
Hi  greeter => '9861675f8845' : 10
OKD  greeter => '9861675f8845' : 100
```

- Let's now delete the greeter configurations. First, stop the loop in the other terminal. The execute the following:

```
oc delete -f 14-delete-configuration.yaml
15-delete-route.yaml
```

- Let's now demo autoscaling. Create the greeter service again.

```
oc apply -f 16-autoscale.yaml
```

- Wait for the service to scale to zero.

- Once the service has no pods, run the following command to performance test and demonstrate the autoscaling:

```
bash 17-perf-test.sh
```

Notice that the number of pods has increased from zero to many. 

![/assets/img/2019-08-21-introduction-to-knative/autoscale.png](/assets/img/2019-08-21-introduction-to-knative/autoscale.png)
- Now delete this service so we can go on to the next demo.

```
oc delete -f 18-delete-autoscale.yaml
```

- Deploy the sink service:

```
oc apply -f 19-deploy-sink-service.yaml
```

- Deploy a channel:

```
oc apply -f 20-channel.yaml
```

- Deploy a cronjob event source that will send '{"message": "Thanks for using OKD"}' to the channel:

```
oc apply -f 21-event-source.yaml
```

- Subscribe the event-greeter to the channel. 

```
oc apply -f 22-channel-subscriber.yaml
```

Notice that the event-greeter pod will scale to 1 after some time. And if you look at the logs of the event-greeter, you will see something like:

![/assets/img/2019-08-21-introduction-to-knative/subscription.png](/assets/img/2019-08-21-introduction-to-knative/subscription.png)

