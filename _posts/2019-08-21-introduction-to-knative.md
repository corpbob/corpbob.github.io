---
layout: post
title: "Introduction to Serverless using Knative"
author: "Bobby Corpus"
categories: journal
tags: [documentation,sample]
image: 2019-08-21-introduction-to-knative/ExampleModel.png-cards2.png
---

# Assumptions

- You have OKD 3.11 up and running

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

- Create the demo 

```
oc apply -f channel.yaml.orig 
oc apply -f event-source.yaml
oc apply -f channel-subscriber.yaml
```
