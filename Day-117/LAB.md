# Day 117 — Lab: Model Serving & Kubeflow

**Goal:** Deploy a Python ML model via KServe and set up A/B testing between two model versions — the assigned hands-on activity for today, broken into concrete steps.

**Prerequisites:** A Kubernetes cluster with KServe installed (via the quickstart, which also installs Knative Serving and Istio as dependencies — a `kind`/`minikube` cluster with adequate resources works for the CPU-only sklearn example below). `kubectl` configured. A trained model artifact (reuse Day 116's `breast-cancer-classifier` from MLflow, exported/copied to a plain S3 path KServe can read, or train a fresh throwaway sklearn model and `joblib.dump`/`mlflow.sklearn.save_model` it to a local/S3 path).

---

### Lab 1 — Install KServe (quickstart mode)

1. Follow the KServe quickstart to install into a local cluster (this installs a minimal Knative + Istio + KServe stack suitable for a lab, not a production topology):
   ```bash
   curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.13/hack/quick_install.sh" | bash
   ```
2. Confirm the controller is running:
   ```bash
   kubectl get pods -n kserve
   ```

**Success criteria:** `kubectl get pods -n kserve` shows the `kserve-controller-manager` pod `Running`.

---

### Lab 2 — Deploy a first model version

1. Save a trained sklearn model in KServe's expected directory layout and upload it somewhere KServe can read (a local MinIO bucket, or the public KServe example model for a first smoke test):
   ```yaml
   apiVersion: serving.kserve.io/v1beta1
   kind: InferenceService
   metadata:
     name: sklearn-iris
     namespace: default
   spec:
     predictor:
       sklearn:
         storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"   # public example model
   ```
2. Apply it and wait for it to become ready:
   ```bash
   kubectl apply -f isvc-v1.yaml
   kubectl get inferenceservice sklearn-iris -w
   ```
3. Send a test prediction request (adjust host/port for your local ingress setup per the KServe quickstart docs):
   ```bash
   curl -v -H "Host: sklearn-iris.default.example.com" \
     http://localhost:8080/v1/models/sklearn-iris:predict \
     -d '{"instances": [[6.8, 2.8, 4.8, 1.4]]}'
   ```

**Success criteria:** You receive a JSON prediction response, and you can explain what `storageUri` pointed at and why the response came back framework-agnostically (same `/v1/models/...:predict` shape regardless of sklearn vs. any other framework).

---

### Lab 3 — The core hands-on activity: deploy a second version and A/B test

1. Train (or reuse) a second model variant with different hyperparameters, save it to a second `storageUri` path.
2. Update the `InferenceService` to add a canary revision at a fixed traffic split:
   ```yaml
   apiVersion: serving.kserve.io/v1beta1
   kind: InferenceService
   metadata: {name: sklearn-iris, namespace: default}
   spec:
     predictor:
       canaryTrafficPercent: 20
       sklearn:
         storageUri: "s3://my-bucket/models/sklearn-iris-v2"
   ```
3. Apply the update, then send 20-30 requests in a loop and log which revision served each (check response metadata / `ce-*` headers, or add a distinguishing dummy field to each model's output if needed) to empirically confirm roughly an 80/20 split:
   ```bash
   for i in $(seq 1 30); do
     curl -s -H "Host: sklearn-iris.default.example.com" \
       http://localhost:8080/v1/models/sklearn-iris:predict \
       -d '{"instances": [[6.8, 2.8, 4.8, 1.4]]}'
   done
   ```
4. Promote the challenger to 100% once satisfied, by removing the canary block and pointing `storageUri` fully at the new model.

**Success criteria:** You've observed both revisions receiving traffic roughly proportional to the configured split, and you can explain in one sentence how this differs from a full blue/green cutover.

---

### Lab 4 — GPU scheduling awareness (conceptual if no GPU available)

1. If you have GPU-enabled nodes available (cloud or workstation with NVIDIA GPU + device plugin installed), request one explicitly:
   ```yaml
   resources:
     limits: {nvidia.com/gpu: 1}
   ```
2. If not, simulate the exercise: write out what taint/toleration pair you'd add to keep non-GPU pods off a GPU node pool, and what `nodeSelector`/`requirements` you'd add to a Karpenter NodePool restricting it to GPU instance types (`g5.*`).

**Success criteria:** You can write, from memory, a correct taint + toleration pair for isolating GPU nodes, even without a physical GPU to test against.

---

### Cleanup

```bash
kubectl delete inferenceservice sklearn-iris -n default
# If you installed the full quickstart stack purely for this lab and want to tear it down:
# follow the KServe quickstart's uninstall instructions for your specific install method.
```

### Stretch challenge

Configure KServe's built-in traffic-splitting to route based on a request header (e.g., `X-Canary: true` always hits the challenger) instead of a fixed percentage, using an Istio `VirtualService` you write yourself layered on top of the two `InferenceService` revisions. This is closer to how a real org lets internal testers dogfood a challenger model before it's exposed to any real percentage of production traffic.
