### Install AWS Storage controller for our PVC's

```bash
$ helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
$ helm repo update
$ helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver --namespace kube-system
```

### Create default storage class
```bash
$ kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
$ kubectl get storageclass
```

`Note`: Please do not forget to attach the following IAM Policy as the permission to the IAM role of the Node group go give an acces to the CSI driver to get access to create EBS voulmes (required by PVC's).

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

#### To keep logs we can use another ways too which we can get from this link https://airflow.apache.org/docs/helm-chart/stable/manage-logs.html. I have decied Elasticsearch like as the best practise.
Or we can store inside of the PVC like as the following

```bash
helm upgrade --install airflow apache-airflow/airflow \
  --set logs.persistence.enabled=true \
  --set logs.persistence.existingClaim=my-volume-claim
```

#### Easy way to deploy Airflow to our requirements it is to read all default values from this link https://github.com/apache/airflow/blob/main/chart/values.yaml and keep local copy to ovverride them for our requirements, or we can override these values with the `--set` option
```bash
$ helm install airflow apache-airflow/airflow --namespace airflow -f values.yaml
```

```bash
$ export NAMESPACE=airflow RELEASE_NAME=airflow && kubectl create namespace $NAMESPACE
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-connection-secret
  namespace: airflow
type: Opaque
stringData:
  elasticsearch: |-
    {
      "scheme": "http",
      "user": "elastic",
      "pass": "password",
      "host": "elasticsearch-host",
      "port": 9200
    }
EOF
$ helm repo add apache-airflow https://airflow.apache.org
$ helm upgrade --install $RELEASE_NAME apache-airflow/airflow \
  --set executor=KubernetesExecutor \
  --set elasticsearch.enabled=true \
  --set elasticsearch.secretName=elasticsearch-connection-secret \
  --namespace $NAMESPACE \
  --create-namespace
```

## Method 1: Using a Volume to Store DAGs

### Create a PersistentVolumeClaim (PVC) for DAGs:
```bash
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-dags-pvc
  namespace: airflow
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

### Update `values.yaml` to include the DAGs volume:
```bash
dags:
  path: /opt/airflow/dags
  persistence:
    enabled: true
    existingClaim: airflow-dags-pvc
```

### After deployment of the Aifrlow, Deploy DAGs by copying them to the PVC:

```bash
$ kubectl cp <local-dag-directory> <your-pod-name>:/opt/airflow/dags
```

## Method 2: Using a Custom Docker Image

### Create a Dockerfile to include your DAGs:

```bash
FROM apache/airflow:2.4.1
COPY dags /opt/airflow/dags
```

### Build and push the image:

```bash
$ docker build -t <your-repo>/airflow:latest .
$ docker push <your-repo>/airflow:latest
```

### Update `values.yaml` to use the new image and deploy airflow:
```bash
images:
  airflow:
    repository: <your-repo>/airflow
    tag: latest
```

```bash
$ helm upgrade airflow apache-airflow/airflow --namespace airflow -f values.yaml
```

## Method 3: Using Git-Sync Sidecar Container

### Update `values.yaml` to configure `Git-Sync` and deploy airflow:
```bash
dags:
  gitSync:
    enabled: true
    repo: "https://github.com/your-repo/your-dags.git"
    branch: "main"
    interval: 60
```

```bash
$ helm upgrade airflow apache-airflow/airflow --namespace airflow -f values.yaml
```

`Node`: I prefer to use it from PVC because it is easy way to copy DAGs to the POD. But if we use it from CI/CD pipelines then, automatically syncs DAGs from a Git repository will be useful. 







