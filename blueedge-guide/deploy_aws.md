# AWS Cloud Deployment Guide

The BlueEdge system can be deployed to the AWS cloud with EKS 

## Install required tools

Install the following CLI tools to your system:

- AWS cli tool: To login and create AWS services
- `kubectl`: To connect to the Kubernetes cluster and control the services.
- `pulumi`: Deploy the system using Python (Infrastructure as code)

## Prepare the accounts

Prepare the AWS IAM account with at least have the following permissions:

- Create/update VPC
- Create/update IAM
- Create EKS Kubernetes Cluster
- Create/update MSK (Kafka) 
- Create/update ElastiCache (Redis)
- Create/update EC2 Instance
- Create/update Auto Scaling Group.
- Create/update ELB
- Create/update Cloudwatch

Also, register a Pulumi account to be able to execute the Pulumi commands.

### Login to AWS CLI

Run `aws configure` to set up the AWS CLI tool, and follow the instruction.

### Login to pulumi

Run `pulumi login` to login to the Pulumi CLI Tool.

## Modify the system configs

The configs for the system is available at `blueedge/deployment/pulumi/system/Pulumi.dev.yaml`. The command to export those configs are listed in the `blueedge/deployment/pulumi/system/README.md` file.

There are a few configs that you may need to pay more attention to:

- `aws:region`: The AWS Region to deploy to services to. If changed, many region specific services and resources may need to be changed as well, e.g. `AMI IDs`.
- `cpu_ami_id`: AMI ID for CPU instances which will host the BlueEdge services. Should be changed according to the selected region, and the AMI should already contains the required Docker Images.
- `gpu_instance_ami_id`: AMI Id for GPU instances which will host the engines which use GPU.
- `mongo_connection_str` and `sentry_endpoint` are the secrets. After login to a new Pulumi account, you must re-run the export command to reset the secret, e.g. `pulumi config set --secret --path 'blueedge.mongo_connection_str' 'mongodb+srv://admin:pAsSwoRd@cluster0.htalq.mongodb.net/?retryWrites=true&w=majority'`
- `allow_ssh_access`: if set to `true`, the EC2 instance will be deployed to Public subnet and have public IP so you can `ssh` to. The ssh key name is defined with `ssh_key_name` value.

## Modify the BlueEdge service configs

The configs for the system is available at `blueedge/deployment/pulumi/blueedge-manager/Pulumi.dev.yaml`. Mind the `ai_endpoint` and `endpoint` values, which should match the endpoints for each environments.

## Start the system

All deployment code are available at `blueedge/deployment/pulumi`. In the directory there are 3 folders:

- `system`: Contains the definintions of AWS and K8S infrastructure components.
- `blueedge-manager`: Contains the definitions of key BlueEdge Services.
- `monitor`: Contains the services for monitoring the system.

Run or follow the commands in the `deploy_all.sh` file to start the system:

```bash
cd blueedge/deployment/pulumi
./deploy_all.sh
```

Or:

```bash
cd blueedge/deployment/pulumi

# ======
# Deploy the system infrastructure first
cd system
pulumi up # Run and wait for the components to be Ready

# Export the kubeconfig, so the kubectl command can know how to connect to the kubernetes cluster
pulumi stack output kubeconfig > kubeconfig.json
export KUBECONFIG=~/Dropbox/SETA/core/blueedge/deployment/pulumi/system/kubeconfig.json

# Check the kubectl has been success fully connected
kubectl get node # should see the node in the cluster

# Install the KEDA autoscaling plugin
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

# Start the BlueEdge manager services
cd ../blueedge-manager
pulumi up

# Start the BlueEdge monitoring services
cd ../monitor/monitoring-stack
helm dependency update
helm upgrade --install blueedge-monitoring-stack .
```

### Access the APIs

To get the Service Endpoints, run:

```bash
kubectl get svc
```

You should see the list of all Services in the K8S cluster, and two of them have type LoadBalancer and have a public endpoint which you can call. One is the endpoint to call the BlueEdge Restful service, and the other is the endpoint of Grafana app. Use those endpoints to access the services.

## Deploy the engines

Make sure the engine has been Dockerized and deployed to the ECR. You can write a `deployment.yml` file which define K8S Deployment for the engine, as well as the scale configuration for the engine. E.g:

> Mind the engine ID in the `ScaledObject` definition. It must match the deployed engine's ID.

```yml
# Engine Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blueengine-car-damage-detect-v1
  labels:
    app: blueengine-car-damage-detect
spec:
  selector:
    matchLabels:
      app: blueengine-car-damage-detect
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: blueengine-car-damage-detect
    spec:
      nodeSelector:
        workload: cuda-required
      containers:
      - name: car-damage-segmentation-ctn
        image: 362085406009.dkr.ecr.us-east-2.amazonaws.com/car-damage-segmentation:1.7.4 # This is the image ECR Uri.
        resources:
          requests:
            memory: "30Gi"
            cpu: "7"
            nvidia.com/gpu: "1"
          limits:
            memory: "30Gi"
            cpu: "7"
            nvidia.com/gpu: "1"
        volumeMounts:
        - mountPath: "/config/system_comm"
          name: comm-config-vol
        - mountPath: "/config/engine"
          name: car-damage-segmentation-config-vol
        env:
        - name: BLUEEDGE_CFG
          value: "/config/engine/cfg.yml"
        - name: BLUEEDGE_COMM_CONFIG
          value: "/config/system_comm/system-comm.yml"
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: blueedge-system-configs
              key: REDIS_HOST
        - name: REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: blueedge-system-configs
              key: REDIS_PORT
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blueedge-system-configs
              key: REDIS_PASSWORD
      volumes:
      - name: comm-config-vol
        secret:
          secretName: blueedge-system-configs
      - name: car-damage-segmentation-config-vol
        configMap:
          name: blueengine-car-damage-segmentation-cfg
---
# Additional Configuration for the engine
apiVersion: v1
kind: ConfigMap
metadata:
  name: blueengine-car-damage-segmentation-cfg
data:
  cfg.yml: |
    ENGINE:
      DEFAULT_WORKER_COUNT: 3
      MAX_WORKER: 3
      KAFKA_CONSUMER_POOL: 1
      KAFKA_PRODUCER_POOL: 1
    CLIENT:
      WORKER_INACTIVE_TIMEOUT: 100.0
    LOG:
      STDOUT_LEVEL: "TRACE"

---
# Scaling config of the engine
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: scaled-blueengine-car-damage-detect-v1
spec:
  scaleTargetRef:
    apiVersion:    apps/v1
    kind:          Deployment
    name:          blueengine-car-damage-detect-v1
    # envSourceContainerName: .spec.template.spec.containers[0]
  pollingInterval: 1
  cooldownPeriod:  60
  minReplicaCount: 8
  maxReplicaCount: 8

  advanced:                                          # Optional. Section to specify advanced options
    restoreToOriginalReplicaCount: true       # Optional. Default: false
    horizontalPodAutoscalerConfig:                   # Optional. Section to specify HPA related options
      behavior:                                      # Optional. Use to modify HPA's scaling behavior
        scaleDown:
          stabilizationWindowSeconds: 60
          policies:
          - type: Percent
            value: 50
            periodSeconds: 2
  triggers:
    - type: redis
      metadata:
        hostFromEnv: REDIS_HOST
        portFromEnv: REDIS_PORT
        passwordFromEnv: REDIS_PASSWORD
        listName: __engine_pending_list_sSzMfsnBpWVLrhUHw2HP3__ # sSzMfsnBpWVLrhUHw2HP3 is the engine's ID
        listLength: "5" # Required
        enableTLS: "false" # optional
        databaseIndex: "0" # optional
---
```

Run `kubectl apply -f deployment.yml` to deploy the engine.

## Destroy the system

To destroy the system, first you need to destroy all engine deployments. Otherwise, the engine Docker container will hold a Network Interface which will prevent the VPC to be fully destroyed.

Run `destroy_all.sh` to destroy the system. If there are any problem destroying the system, you can go to AWS console to manually destroy the resources and re-run `pulumi destroy` in the `system` dir, so the pulumi app can update the status. 