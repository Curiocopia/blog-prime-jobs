# 📦 Prime Jobs

This is an adaptation of a sample Kubernetes docs learning example ([fine parallel processing work queue]). Following the instructions of the document a Job is deployed to process natural numbers placed in a Redis queue to check for a primality related test. Details are described in [Prime Jobs].

## 🌟 Highlights

- Uses Redis work queue for work scheduling 
- Communicates (via requests) with a backend-service that uses [SageMath] for computations 

## ℹ️ Overview

Please refer to [Prime Jobs] for the specific blog and the relevant repo(s). 

### ✍️ Authors

All repos shared in [CurioCopia] are shared under Creative Commons license for others to adopt and use it as they wish.

## 🚀 Usage

1. Generate your own Docker image for [oppermann-worker], place it in your favorite registry.
2. Generate your own Docker image for [sagemath-backend-service]. Instantiate it in a compute accessible from the Kubernetes cluster where `prime-job` will be running.
3. Clone this repo, adjust the essential parameters in [demo], run kustomize and generate resource files.
4. Create the Redis Pod and Service as well as headless Sagemath Service and EndpointSlices.
5. Once the Redis Service is available, run the `mo-fill-job` to store the numbers to be processed.
6. Run the `prime-job` to process the numbers placed in the Redis queue.
7. Collect the logs of the Pods created for the Job execution.
```shell-session
kubectl get po
NAME                READY   STATUS      RESTARTS       AGE
prime-job-qsgs9      0/1     Completed   0              79s
prime-job-z6gmq      0/1     Completed   0              79s
mo-fill-job-b6k94   0/1     Completed   0              2m20s
redis-master        1/1     Running     0              4d20h
temp                1/1     Running     1 (3h5m ago)   3h20m
kubectl logs prime-job-qsgs9
Worker with sessionID: e6c62a6d-0df1-437c-8e57-f4b3f8712741
Initial queue state: empty=False
Waiting for work
Queue empty, exiting
kubectl logs prime-job-z6gmq
Worker with sessionID: d283ac4e-ddaf-4b47-a396-e261cef8ee91
Initial queue state: empty=False
1000132  yields a MAGIC with a pair of  36055  prime numbers on either side of its square.
Queue empty, exiting
```

## ⬇️ Installation

Let's follow the typical Kustomize installation process.

Define a place to work:
```bash
DEMO_HOME=$(mktemp -d)
```
### Establish the Base

```bash
BASE=$DEMO_HOME/base
mkdir -p $BASE

CONTENT="https://raw.githubusercontent.com/curiocopia/blog-prime-jobs"

curl -s -o "$BASE/#1" "$CONTENT/base\
/{externalsagemath-endpointslice.yaml,externalsagemath-service.yaml,kustomization.yaml, mo-fill-job.yaml,prime-job-yaml,redis-pod.yaml,redis-service.yaml,prime-job.env}"
```
Look at the directory:
```bash
tree $DEMO_HOME
```
Expect something like:
```bash
/tmp/tmp.OdCWAqtRU4
└── base
    ├── externalsagemath-endpointslice.yaml
    ├── externalsagemath-service.yaml
    ├── kustomization.yaml
    ├── mo-fill-job.yaml
    ├── prime-job.env
    ├── prime-job.yaml
    ├── redis-pod.yaml
    └── redis-service.yaml
```
### The Base Customization

The base directory has a kustomization file:
```bash
more $BASE/kustomization.yaml
```
You can run kustomize on the base to emit customized resources to stdout and inspect:
```bash
kustomize build $BASE
```
More conveniently you can generate the kustomize output and split the resources into their own files and then execute them in the preferred order.
```bash
RESOURCES=$DEMO_HOME/resources
mkdir -p $RESOURCES

kustomize build $BASE -o $RESOURCES

tree $DEMO_HOME
/tmp/tmp.OdCWAqtRU4
├── base
│   ├── externalsagemath-endpointslice.yaml
│   ├── externalsagemath-service.yaml
│   ├── kustomization.yaml
│   ├── mo-fill-job.yaml
│   ├── prime-job.env
│   ├── prime-job.yaml
│   ├── redis-pod.yaml
│   └── redis-service.yaml
└── resources
    ├── batch_v1_job_mo-fill-job.yaml
    ├── batch_v1_job_prime-job.yaml
    ├── discovery.k8s.io_v1_endpointslice_sagemath-service-endpointslice.yaml
    ├── v1_configmap_prime-job-environment-vars-hf52cf99mh.yaml
    ├── v1_pod_redis-master.yaml
    ├── v1_service_redis.yaml
    └── v1_service_sagemath-service.yaml
```
Follow the recipe below (adjust for your own exact filenames) for ConfigMap, Redis Pod and Service and headless SageMath Service and EndPointSlice:
```bash
cd $RESOURCES

kubectl create -f v1_configmap_prime-job-environment-vars-hf52cf99mh.yaml
kubectl create -f v1_pod_redis-master.yaml
kubectl create -f v1_service_redis.yaml
kubectl create -f v1_service_sagemath-service.yaml
kubectl create -f discovery.k8s.io_v1_endpointslice_sagemath-service-endpointslice.yaml
```
Once the resources are running, follow the recipe below for filling the Redis queue and processing it.
```bash
kubectl apply -f batch_v1_job_mo-fill-job.yaml
kubectl wait --for=condition=complete job/mo-fill-job --timeout=60s
kubectl apply -f batch_v1_job_prime-job.yaml
```

## 💭 Feedback and Contributing

As desribed in [Prime Jobs], so far the largest magic Oppermann number found is 1000132. Once you find a larger number please share it in Discussions.

If you have any other suggestions for improvements or corrections, please drop a note in Discussions.

[fine parallel processing work queue]: https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/
[Prime Jobs]: https://curiocopia.com/blog/prime-jobs
[SageMath]: https://sagemath.org
[Curiocopia]: https://curiocopia.com
[oppermann-worker]: https://github.com/Curiocopia/oppermann-worker
[sagemath-backend-service]: https://github.com/Curiocopia/SageMath-backend-service
[demo]: overlays/demo/