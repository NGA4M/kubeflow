# NGA4M

This is an MLOps platform that follows a multicluster architecture. In fact, the platform consists of a Primary Cluster(PC) and several Secondary Clusters(SC). 

## Description

The PC is composed of the deployment of kubeflow performed by manifest (https://github.com/kubeflow/manifests) and the deployment of the prometheus operator performed by helm chart (https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) .

The SC is composed of the deployment of kserve (https://kserve.github.io/website/master/get_started/#install-the-kubernetes-cli) carried out via installation script and the deployment of the prometheus deployed as above.


## Setup

This section will describe how to perform the setup of the two types of clusters.

### PC Setup

This section will describe how to perform the setup of the Primary cluster.

#### Kubeflow installation

First, the linux kernel sub system needs to be modified to support a large number of pods:

```bash
sudo sysctl fs.inotify.max_user_instances=2280
sudo sysctl fs.inotify.max_user_watches=1255360
```
then, you need to install kustomize with the following command:

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

Then you need to log in with your dockerhub credentials with the command:

```bash
docker login
```
and create a k8s secret with the credentials you just entered with the command :

```bash
kubectl create secret generic regcred \
	--from-file=.dockerconfigjson=$HOME/.docker/config.json \
	--type=kubernetes.io/dockerconfigjson
```
Finally, after cloning the repository that contains the kubeflow manifest:

```bash
git clone https://github.com/kubeflow/manifests.git
```
you need to move to the newly created folder and run the command to apply the yaml:

```bash
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```

#### Enable jupyter to access Kfp

To enable jupyter notebook to use the kubeflow pipelines python SDK, you must apply the following PodDefault by entering the namespace name of your kubeflow profile in the namespace field:

```yaml
apiVersion: kubeflow.org/v1alpha1
kind: PodDefault
metadata:
  name: access-ml-pipeline
  namespace: "<YOUR_USER_PROFILE_NAMESPACE>"
spec:
  desc: Allow access to Kubeflow Pipelines
  selector:
    matchLabels:
      access-ml-pipeline: "true"
  env:
    - ## this environment variable is automatically read by `kfp.Client()`
      ## this is the default value, but we show it here for clarity
      name: KF_PIPELINES_SA_TOKEN_PATH
      value: /var/run/secrets/kubeflow/pipelines/token
  volumes:
    - name: volume-kf-pipeline-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 7200
              ## defined by the `TOKEN_REVIEW_AUDIENCE` environment variable on the `ml-pipeline` deployment
              audience: pipelines.kubeflow.org      
  volumeMounts:
    - mountPath: /var/run/secrets/kubeflow/pipelines
      name: volume-kf-pipeline-token
      readOnly: true

```

The PodDefualt is a CRD used to automatically enrich Pods with default configurations such as environment variables, volumes, or other settings.

Once this is done, when creating a new notebook, you need to tick this box:

![alt text](/img/config.png)

#### MLFlow integration

This section will discuss how to integrate MLFow as a model registry within kubeflow. In this case we will deploy mlflow along with an instance of mysql used as a metadata store and an instance of minio used as an artifact store.


First we need to apply all the yaml files present in the ```mlflow/mysql_mlflow``` folder to deploy a mysql instance. They are present in the order: 

- the yaml file to create the secret with the login credentials for the database, 

- the yaml file to create the pvc that serves as storage for the mysql instance, 

- and finally the yaml file with the deployment and service of mysql.


Next you need to apply all the yaml in the ```mlflow/minio-mlflow``` folder, in that order: 

- the yaml file to create the secret with the minio login credentials, 

- the yaml file to create the pvc that serves as storage for the minio instance, 

- and finally the yaml file with the minio deployment and service.

Once you have deployed mysql and minio, you need to apply the yaml files present in ```mlflow``` directory, in order:

- the yaml file for the mlflow deployment, 
- the yaml file for the service and for the virtualservice that will be used to add it to the kubeflow dashboard. Its function is to manage and control network traffic between services in the cluster.

To add mlflow to the kubeflow dashboard, you need to edit the configmap of the kubeflow manifest through the ```kustomize``` tool. The configmap to edit is located at path ```manifests/apps/centraldashboard/upstream/overlays/kserve/patches/configmap.yaml```

this, should be modified by adding the following:

```
{
   "link": "/mlflow/",
   "text": "MLFlow",
   "type": "item"
},
```
Once the file has been updated, you can apply the changes with the following command

```bash
kustomize build apps/centraldashboard/overlays/oauth2-proxy | kubectl apply -f -
```

#### Prometheus Operator installation

To install the prom-operator helm is used; first the repository must be added and updated:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Next, you cann instsall the operator with the following command:

```bash
helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack
```

### SC Setup

This section will explain how to perform Secondary Cluster setup. After installing the prom-operator, as mentioned in the previous section, we need to install kserve.

#### Kserve Deployment


To deploy kserve in the secondary clsuter, you need to run the following command 

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.14/hack/quick_install.sh" | bash
```

### Object Storage setup

You also need to have an object storage that is accessible to all clusters (PCs and all SCs). In this specific case, a MinIo instance is used

## Full Example

Below is a complete example of using the platform based on a neural network that classifies the iris dataset.

### Pipeline creation using KFP

The first component we’ll cover is kubeflow pipelines, an open-source platform for orchestrating machine learning (ML) workflows on Kubernetes, allowing you to automate, monitor, and manage scalable, reusable ML pipelines.

Here is the link to the python sdk documentation. https://kubeflow-pipelines.readthedocs.io/en/sdk-2.11.0/

The pipeline will cover all the phases of the process, from reading and preprocessing the dataset, to serving the trained model. 

Briefly, the pipeline includes a first step of reading the dataset from the shared object storage, a second step of preprocessing, a third step of creating the model in TensorFlow, a fifth step of training, a sixth step of converting the model into onnx format and loading it into the shared object storage, and a seventh and final step of serving the model into one of the secondary clusters.

Below is an image of the complete pipeline schematic

![alt text](/img/pipeline.png)

The pipeline was created using the "notebook" functionality provided by kubeflow and using the KFP python SDK. In reality, it is not strictly necessary to use kubeflow notebooks to create the pipeline; you can in fact use the SDK outside of kubeflow to generate a yaml file that will then be used to generate the pipeline thanks to kubeflow UI. The advantage of using kubeflow notebooks is that you can directly run the pipeline thanks to a python function.

```Full_example/iris_pipeline.ipynb``` file  contains the jupyter notebook with the code of the whole pipeline and the ```Full_example/iris.csv``` file contains the dataset on which to train the network.

The jupyter notebook, if executed, generates a yaml file that, if submitted via kubeflow UI, allows the pipeline to be executed.

to ensure that containers generated by the pipeline have access to the MinIO credentials, you need to create a secret with the credentials inside the kubeflow user namespace. A secret template is provided at ```Full_example/secret_example.yaml```. Thanks to the "kfp.kubernetes" library, the secret will then be mounted inside the pods of the pipeline steps that need it as an environment variable (or possibly as a file). 


### Serving using Kserve

The last step of the pipeline is to serve the model to one of the secondary clusters. The first thing to make sure is that the designated cluster has a ServiceAccount linked to a secret, which allows the cluster to access the shared object storage to download the model. A template yaml file to create the Service Account and secret is provided in ```Full_example/service_account_example.yaml``` 

Another fundamental thing to make multicluster serving work is to have the kubeconfig file, which provides the keys and permissions to deploy the model on a specific cluster. In the provided configuration, the file in question is located inside the object storage. A more in-depth look at the hierarchy of files in object storage will follow.


### Object storage hierarchy

In this configuration, a particular file hierarchy has been applied inside the object storage. Some choices have been forced by some constraints imposed by the triton inference service that needs a particular file organization to download the model to be served correctly.

the proposed hierarchy is as follows:

``` plain text
iris-bucket
    └── iris_example
        ├── config
        ├── iris
        │   ├── 1
        │   │   └── model.onnx
        │   └── config.pbtxt
        └── IRIS.csv
```
The files to insert manually are ```IRIS.csv``` and ```config``` (the kubeconfig file). It was thought to insert them in the object storage to make it easier for the pipeline to retrieve them, but they could, for example, be mounted via volumes inside the pipeline pods. Starting from the ```iris``` folder level, everything is created automatically by the pipeline and the file hierarchy cannot be changed because it is required by kserve to work correctly.


### Monitoring with Prometheus

To monitor models with prometheus we will use a federated setup. In this setup we have the secondary cluster prometheus collecting metrics from the model pod and sending them to the primary cluster prometheus.

The first thing to do, in this case, is to tell the secondary cluster's prometheus to collect the model metrics. To do this, we must first add, in the serving phase, the annotation ```'serving.kserve.io/enable-prometheus-scraping': 'true'``` that allows the model to expose the metrics (this is done in ```Full_example/iris_pipeline.ipynb```). 

Next, we will need to deploy a service that allows prometheus to access the pods port 8002, which is where the metrics are exposed. The service yaml file is present in ```Full_example/prometheus_service.yaml```.

Finally, you need to deploy a ServiceMonitor that has the task of adding the model pods to the targets that promtheus needs to monitor. A template is present in ```Full_example/servicemonitor.yaml```.

To allow the primary cluster Prometheus to collect the metrics sent by the secondary cluster Prometheus, you need to update the primary cluster Prometheus helm chart, changing the default values, adding the secondary Prometheus endpoint that will send the metrics. 

To do this, you need to run the following command: ```helm upgrade [prometheus release name] prometheus-community/kube-prometheus-stack -n [prometheus namespace] -f values.yaml```

The values.yaml file can be found from the official prometheus chart repository https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml.

Precisely you need to modify the "additionalScrapeConfigs" section for example like this:

```yaml
additionalScrapeConfigs:
    - job_name: 'federate'  #insert job name as you wish
      scrape_interval: 30s

      honor_labels: true
      metrics_path: '/federate'

      params:
        'match[]':
          - '{namespace="modelli"}'  #insert ml models namespace

      static_configs:
        - targets:
          - 'prometheus.mm'  #insert the secondary cluster's prometheus endpoint
```

Obviously the secondary Prometheus will have to be reachable from the primary Prometheus.


### Grafana dashboard

The ```Full_example/deashboard.json``` file contains a Grafana dashboard to monitor models being served in secondary clusters.

![alt text](/img/grafana.png)


### Inference

At ```inference/iris_inference.ipynb``` there is a complete example of how to do inference on the iris model with the kserve v2 protocol

### Pipeline's docker image

To recreate the docker images that the pipeline containers are based on, all the files are listed in ```docker_images``` directory