# Training MNIST using Kubeflow, S3, and Argo.

This example guides you through the process of taking an example model, modifying it to run better within Kubeflow, and serving the resulting trained model. We will be using Argo to manage the workflow, Tensorflow's S3 support for saving model training info, Tensorboard to visualize the training, and Kubeflow to deploy the Tensorflow operator and serve the model.

## Prerequisites

Before we get started there a few requirements.

## 1.Kubernetes Cluster Environment

* ICP v3.1.0/v3.1.1
* Kubernetes v1.11.1/v1.11.3

## 2.Setup Kubeflow

### Download ksonnet

```
curl -fksSL https://github.com/ksonnet/ksonnet/releases/download/v0.13.0/ks_0.13.0_linux_amd64.tar.gz | tar --strip-components=1 -xvz -C /usr/local/bin/ ks_0.13.0_linux_amd64/ks
```

### Install kubeflow

```
git clone -b v0.3.2 https://github.com/kubeflow/kubeflow
./kubeflow/scripts/kfctl.sh init kf-app --platform none
cd kf-app
../kubeflow/scripts/kfctl.sh generate k8s
../kubeflow/scripts/kfctl.sh apply k8s
```

### Create PV for vizier-db

This example is using Kubernetes local storage, please change the ip address `172.16.183.209` to the value based on your environment.

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vizier-db
spec:
  capacity:
    storage: 40Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /var/pv/vizier-db
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - 172.16.250.140
```

## 3.Setup Minio

### Deploy Minio

```
docker run --name=minio --net=host -e MINIO_ACCESS_KEY=minio -e MINIO_SECRET_KEY=minio123 -d -v /var/lib/minio:/data siji/minio server /data
```

The Minio access key is `minio`, secret key is `minio123`.


### Create Minio bucket

```
mkdir /var/lib/minio/tfmnist
```

**OR**

Open Minio dashbaord in browser: http://minio-host:9000

In my case, it is http://9.30.218.75:9000

Create bucket: `tfmnist`

**NOTE**: Create different bucket for different training. If you do not want to persist the old training data, just use one bucket and the new data will overwrite the old data.


## 4.Download argo

```
curl -sSL -o /usr/local/bin/argo https://github.com/argoproj/argo/releases/download/v2.2.1/argo-linux-amd64
chmod +x /usr/local/bin/argo
```

## 5.Create tf service account

We will use `mnist` namespace for mnist model.

```
kubectl create ns mnist
kubectl -n mnist apply -f tf-user.yaml
```

## 6.Create secret for workflow

**NOTE**: Change the `S3_ENDPOINT` to your minio endpoint.

```
export NAMESPACE=mnist

export S3_ENDPOINT=9.30.218.75:9000
export AWS_ENDPOINT_URL=http://${S3_ENDPOINT}
export AWS_ACCESS_KEY_ID=minio
export AWS_SECRET_ACCESS_KEY=minio123
export AWS_REGION=us-west-2
export BUCKET_NAME=tfmnist
export S3_USE_HTTPS=0
export S3_VERIFY_SSL=0

kubectl -n ${NAMESPACE} create secret generic aws-creds --from-literal=awsAccessKeyID=${AWS_ACCESS_KEY_ID} \
 --from-literal=awsSecretAccessKey=${AWS_SECRET_ACCESS_KEY}
```

## 7.Submit training workflow

NOTE: By default this training workflow will enable model serving, you can disable model serving by passing `-p model-serving=false` to this workflow. And follow step 8 to enable model serving.

```
export S3_DATA_URL=s3://${BUCKET_NAME}/data/mnist/
export S3_TRAIN_BASE_URL=s3://${BUCKET_NAME}/models
export JOB_NAME=myjob-mnist
export TF_MODEL_IMAGE=siji/mnist-model:v1.11.0
export TF_WORKER=3
export MODEL_TRAIN_STEPS=200

argo submit model-train.yaml -n ${NAMESPACE} --serviceaccount tf-user \
    -p aws-endpoint-url=${AWS_ENDPOINT_URL} \
    -p s3-endpoint=${S3_ENDPOINT} \
    -p aws-region=${AWS_REGION} \
    -p tf-model-image=${TF_MODEL_IMAGE} \
    -p s3-data-url=${S3_DATA_URL} \
    -p s3-train-base-url=${S3_TRAIN_BASE_URL} \
    -p job-name=${JOB_NAME} \
    -p tf-worker=${TF_WORKER} \
    -p model-train-steps=${MODEL_TRAIN_STEPS} \
    -p s3-use-https=${S3_USE_HTTPS} \
    -p s3-verify-ssl=${S3_VERIFY_SSL} \
    -p namespace=${NAMESPACE}
```

Your training workflow should now be executing.

You can verify and keep track of your workflow using the argo commands:

```
$ argo -n ${NAMESPACE} list
NAME                STATUS    AGE   DURATION
tf-workflow-h7hwh   Running   1h    1h

$ argo -n ${NAMESPACE} get tf-workflow-h7hwh
```

After the STATUS to `Succeeded`, then you can use it.


## 8.Submit serving workflow[optional]

**NOTE: Please only run this when you disable model serving in step 7.**

```
argo -n ${NAMESPACE} list # get the workflow name
WORKFLOW=<the workflow name>
argo submit model-deploy.yaml -n ${NAMESPACE} -p workflow=${WORKFLOW} --serviceaccount=tf-user
```

## 9.Using Tensorflow serving

### Install client requirements

```
apt install python-pip python-setuptools --no-install-recommends
pip install -r requirements.txt
```

### Web: Mnist Digit Reader

```
cd mnist-webapp
SERVICE_IP=$(kubectl -n ${NAMESPACE} get service -l app=mnist-${JOB_NAME} -o jsonpath='{.items[0].spec.clusterIP}')
TF_MODEL_SERVER_HOST=$SERVICE_IP python app.py
```

Then open browser with url: http://your-host-ip:5000

In my case, the url is: http://9.30.218.75:5000

### Cli: Submit and query result

By default the workflow deploys our model via Tensorflow Serving. Included in this example is a client that can query your model and provide results:

```
SERVICE_IP=$(kubectl -n ${NAMESPACE} get service -l app=mnist-${JOB_NAME} -o jsonpath='{.items[0].spec.clusterIP}')
TF_MODEL_SERVER_HOST=$SERVICE_IP TF_MNIST_IMAGE_PATH=data/7.png python mnist_client.py
```

This should result in output similar to this, depending on how well your model was trained:
```
outputs {
  key: "classes"
  value {
    dtype: DT_UINT8
    tensor_shape {
      dim {
        size: 1
      }
    }
    int_val: 7
  }
}
outputs {
  key: "predictions"
  value {
    dtype: DT_FLOAT
    tensor_shape {
      dim {
        size: 1
      }
      dim {
        size: 10
      }
    }
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 1.0
    float_val: 0.0
    float_val: 0.0
  }
}


............................
............................
............................
............................
............................
............................
............................
..............@@@@@@........
..........@@@@@@@@@@........
........@@@@@@@@@@@@........
........@@@@@@@@.@@@........
........@@@@....@@@@........
................@@@@........
...............@@@@.........
...............@@@@.........
...............@@@..........
..............@@@@..........
..............@@@...........
.............@@@@...........
.............@@@............
............@@@@............
............@@@.............
............@@@.............
...........@@@..............
..........@@@@..............
..........@@@@..............
..........@@................
............................
Your model says the above number is... 7!
```

You can also omit `TF_MNIST_IMAGE_PATH`, and the client will pick a random number from the mnist test data. Run it repeatedly and see how your model fares!


## 10.Bring JupyterHub up

Change the `ambassador` service type to `NodePort`, then access JupyterHub through ambassador.

```
kubectl -n kubeflow patch service ambassador -p '{"spec": {"type": "NodePort"}}'
kubectl -n kubeflow get service ambassador
```

Then you will find the NodePort and access it by NodePort, use any username and password to login, such as `admin/admin`.

Create a PV in order to bring up a Jupyter NoteBook:

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jupyter-pv
spec:
  capacity:
    storage: 40Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # storageClassName: pets
  local:
    path: /var/pv/jupyter
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - 172.16.250.140
```

## Defining your training workflow

This is the bulk of the work, let's walk through what is needed:

1. Train the model
1. Export the model
1. Serve the model

Now let's look at how this is represented in our [example workflow](model-train.yaml)

The argo workflow can be daunting, but basically our steps above extrapolate as follows:

1. `get-workflow-info`: Generate and set variables for consumption in the rest of the pipeline.
1. `tensorboard`: Tensorboard is spawned, configured to watch the S3 URL for the training output.
1. `train-model`: A TFJob is spawned taking in variables such as number of workers, what path the datasets are at, which model container image, etc. The model is exported at the end.
1. `serve-model`: Optionally, the model is served.

With our workflow defined, we can now execute it.


## Monitoring

There are various ways to monitor workflow/training job. In addition to using `kubectl` to query for the status of `pods`, some basic dashboards are also available.

### Argo UI

The Argo UI is useful for seeing what stage your worfklow is in:

```
PODNAME=$(kubectl -n kubeflow get pod -l app=argo-ui -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward ${PODNAME} 8001:8001
```

You should now be able to visit [http://127.0.0.1:8001](http://127.0.0.1:8001) to see the status of your workflows.

### Tensorboard

Tensorboard is deployed just before training starts. To connect:

```
PODNAME=$(kubectl -n ${NAMESPACE} get pod -l app=tensorboard-${JOB_NAME} -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward ${PODNAME} 6006:6006
```

Tensorboard can now be accessed at [http://127.0.0.1:6006](http://127.0.0.1:6006).


## Submitting new argo jobs

If you want to rerun your workflow from scratch, then you will need to provide a new `job-name` to the argo workflow. For example:

```
#We're re-using previous variables except JOB_NAME
export JOB_NAME=myawesomejob

argo submit model-train.yaml -n ${NAMESPACE} --serviceaccount tf-user \
    -p aws-endpoint-url=${AWS_ENDPOINT_URL} \
    -p s3-endpoint=${S3_ENDPOINT} \
    -p aws-region=${AWS_REGION} \
    -p tf-model-image=${TF_MODEL_IMAGE} \
    -p s3-data-url=${S3_DATA_URL} \
    -p s3-train-base-url=${S3_TRAIN_BASE_URL} \
    -p job-name=${JOB_NAME} \
    -p tf-worker=${TF_WORKER} \
    -p model-train-steps=${MODEL_TRAIN_STEPS} \
    -p s3-use-https=${S3_USE_HTTPS} \
    -p s3-verify-ssl=${S3_VERIFY_SSL} \
    -p namespace=${NAMESPACE}
```

## Conclusion and Next Steps

This is an example of what your machine learning pipeline can look like. Feel free to play with the tunables and see if you can increase your model's accuracy (increasing `model-train-steps` can go a long way).

## Delete

```
../kubeflow/scripts/kfctl.sh delete k8s
```
```
kubectl delete ns mnist
```
