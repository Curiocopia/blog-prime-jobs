# 📦 Prime Jobs

This is an adaptation of a sample Kubernetes docs learning example ([fine parallel processing work queue]). Following the instructions of the document a Job is deployed to process natural numbers placed in a Redis queue to check for a primality related test. Details are described in [Prime Jobs].

## 🌟 Highlights

- Uses Redis work queue for work scheduling 
- Communicates (via requests) with a backend-service that uses [SageMath] for computations 

## ℹ️ Overview

Please refer to [Prime Jobs] for the specific blog and the relevant repo(s). If you don't have time TL;DR:

This is not an effective way to find natural numbers $n$ that satisfy $\pi(n^2)-\pi(n^2-n) = \pi(n^2+n)-\pi(n^2)$ where $\pi(x)$ is the number of prime numbers smaller than or equal to x (as discussed in the blog, called them `magic Oppermann` numbers). Instead it is a practice to use it as an interesting workload in working with the Kubernetes [fine parallel processing work queue] example along with some dabbling in the use of kustomize.

More effective ways could be Jupyter notebooks, Python scripts or SageMath scripts. heck, we could even pass search starting point and search count in the Job definition. However, in that case, the distributed processing tasks would have been significantly lesser in count and boring. Anyway, it was a (personal) choice.

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
prime-job-qsgs9     0/1     Completed   0              79s
prime-job-z6gmq     0/1     Completed   0              79s
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
TEST_HOME=$(mktemp -d)
```
### Establish the Base

```bash
BASE=$TEST_HOME/base
mkdir -p $BASE

CONTENT="https://raw.githubusercontent.com/curiocopia/blog-prime-jobs"

curl -s -o "$BASE/#1" "$CONTENT/base\
/{externalsagemath-endpointslice.yaml,externalsagemath-service.yaml,kustomization.yaml,mo-fill-job.yaml,prime-job.yaml,redis-pod.yaml,redis-service.yaml,prime-job.env}"
```
Look at the directory:
```bash
tree $TEST_HOME
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
RESOURCES=$TEST_HOME/resources
mkdir -p $RESOURCES

kustomize build $BASE -o $RESOURCES

tree $TEST_HOME
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

kubectl apply -f v1_configmap_prime-job-environment-vars-hf52cf99mh.yaml
kubectl apply -f v1_pod_redis-master.yaml
kubectl apply -f v1_service_redis.yaml
kubectl apply -f v1_service_sagemath-service.yaml
kubectl apply -f discovery.k8s.io_v1_endpointslice_sagemath-service-endpointslice.yaml
```
Once the resources are running, follow the recipe below for filling the Redis queue and processing it.
```bash
kubectl apply -f batch_v1_job_mo-fill-job.yaml
kubectl wait --for=condition=complete job/mo-fill-job --timeout=60s
kubectl apply -f batch_v1_job_prime-job.yaml
```
## Create Overlay

Create a `demo` overlay.
```bash
OVERLAYS=$TEST_HOME/overlays
mkdir -p $OVERLAYS/demo
```
## Demo Customization

```bash
curl -s -o "$OVERLAYS/demo/#1" "$CONTENT/overlays/demo\
/{kustomization.yaml,prime-job-patch.yaml,sagemath-endpointslice-patch.yaml,sagemath-service-patch.yaml,prime-job-demo.env}"
```
Adjust the parameters as you need. Set `namespace` for all resources and `prime-job` `image` in `kustomization.yaml`:
```yaml
namespace: demo

images:
- name: oppermann-worker
  newName: my-registry/oppermann-worker
  newTag: v1
```
Adjust `prime-job-demo.env` values for ConfigMap creation to use in various reources.

Change `spec.parallelism` value per your resources in the cluster.
```yaml
spec:
  parallelism: 10 # The new value that you can change.
```
Change the `spec.externalIPs` in the `sagemath-service-patch.yaml` based on the IP address for the `sagemath-backend-service`: 
```yaml
  externalIPs:
  - 192.168.1.200
```
Use the same value for the `endpoints.addresses` in the `sagemath-endpointslice-patch.yaml`: 
```yaml
endpoints:
- addresses:
  - "192.168.1.200"
```
Delete and previous values in the resources and create the new resource files.
```bash
rm $RESOURCES/*
kustomize build $OVERLAYS/demo -o $RESOURCES
```
Inspect the values. If you are satisfied, follow the same base recipe for deployment and execution (in the `kubectl wait` command, do not forget to include the namespace) after you create the `demo` namespace.
## 💭 Feedback and Contributing

As described in [Prime Jobs], so far the largest magic Oppermann number found using this method is 1000132. Once you find a larger number please share it in Discussions.

If you have any other suggestions for improvements or corrections, please drop a note in Discussions.

[fine parallel processing work queue]: https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/
[Prime Jobs]: https://curiocopia.com/blog/prime-jobs
[SageMath]: https://sagemath.org
[Curiocopia]: https://curiocopia.com
[oppermann-worker]: https://github.com/Curiocopia/oppermann-worker
[sagemath-backend-service]: https://github.com/Curiocopia/SageMath-backend-service
[demo]: overlays/demo/