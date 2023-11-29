# k8s-logging-operator-to-es

This is based on guides of [elasticsearch](https://www.elastic.co/guide/en/cloud-on-k8s/2.10/k8s-quickstart.html)
and [logging-operator](https://kube-logging.dev/docs/examples/es-nginx/).

Deviations from the guides:

* logging-operator uses outdated 7.x elastic search
* I use ES operator instead of installing the crds manually.

### Prerequisites

* kubectl
* helm
* minikube
* (recommended) k9s

## Step-by-step

### ECK

```bash
minikube start --memory 8192 --cpus 4
helm repo add elastic https://helm.elastic.co
helm repo update
helm upgrade --install --wait --create-namespace --namespace elastic-system elastic-operator elastic/eck-operator
kubectl create ns logging
kubectl apply -n logging -f elastic.yaml
# Wait until kibana starts, forward the port to log into kibana to check that is works
kubectl port-forward service/quickstart-kb-http 5601 -n logging > /dev/null &
# username: elastic / password: <run command below>
kubectl -n logging get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

### (Optional) Create index for logging-generator.

This removes the duplicate fields xxx.keyword

go to https://localhost:5601/app/dev_tools#/console and run this:

```
PUT /fluentd
{
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
      "agent": {
        "type": "text"
      },
      "asd": {
        "type": "text"
      },
      "code": {
        "type": "text"
      },
      "foo": {
        "type": "text"
      },
      "host": {
        "type": "text"
      },
      "http_x_forwarded_for": {
        "type": "text"
      },
      "kubernetes": {
        "properties": {
          "container_hash": {
            "type": "text"
          },
          "container_image": {
            "type": "text"
          },
          "container_name": {
            "type": "text"
          },
          "docker_id": {
            "type": "text"
          },
          "host": {
            "type": "text"
          },
          "labels": {
            "properties": {
              "app": {
                "properties": {
                  "kubernetes": {
                    "properties": {
                      "io/instance": {
                        "type": "text"
                      },
                      "io/name": {
                        "type": "text"
                      }
                    }
                  }
                }
              },
              "app-kubernetes-io/instance": {
                "type": "text"
              },
              "app-kubernetes-io/name": {
                "type": "text"
              },
              "pod-template-hash": {
                "type": "text"
              }
            }
          },
          "namespace_name": {
            "type": "text"
          },
          "pod_id": {
            "type": "text"
          },
          "pod_name": {
            "type": "text"
          }
        }
      },
      "level": {
        "type": "text"
      },
      "log": {
        "type": "text"
      },
      "method": {
        "type": "text"
      },
      "name": {
        "type": "text"
      },
      "path": {
        "type": "text"
      },
      "referer": {
        "type": "text"
      },
      "remote": {
        "type": "text"
      },
      "size": {
        "type": "text"
      },
      "status": {
        "type": "text"
      },
      "stream": {
        "type": "text"
      },
      "time": {
        "type": "date"
      },
      "title": {
        "type": "text"
      },
      "user": {
        "type": "text"
      }
    }
  }
}
```

### Logging operator

```bash
helm upgrade --install --wait --namespace logging logging-operator oci://ghcr.io/kube-logging/helm-charts/logging-operator
kubectl apply -n logging -f logging.yaml
# wait for deployment
helm upgrade --install --wait --namespace logging log-generator oci://ghcr.io/kube-logging/helm-charts/log-generator
```

The logging generator will send data into index `fluentd`. 
In Kibana create a new dashboard form that index to see the data from logging-generator. 
