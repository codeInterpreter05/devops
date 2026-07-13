# Day 117 — Cheatsheet: Model Serving & Kubeflow

## KServe

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata: {name: my-model, namespace: mlops}
spec:
  predictor:
    sklearn:                      # or: xgboost, pytorch, tensorflow, triton, pmml, paddle, sklearn
      storageUri: "s3://bucket/models/my-model/1"
    minReplicas: 0                # scale to zero when idle
    maxReplicas: 10
    canaryTrafficPercent: 10      # optional: split traffic to the new revision
```

```bash
kubectl apply -f isvc.yaml
kubectl get inferenceservice                          # list, shows READY + URL
kubectl get inferenceservice my-model -o jsonpath='{.status.url}'

# Standard prediction call shape (framework-agnostic)
curl -H "Host: my-model.mlops.example.com" \
  http://<ingress>/v1/models/my-model:predict \
  -d '{"instances": [[...]]}'
```

## Custom Python predictor (when a built-in framework server isn't enough)

```python
import kserve

class MyModel(kserve.Model):
    def __init__(self, name: str):
        super().__init__(name)
        self.load()

    def load(self):
        self.model = load_my_model()
        self.ready = True

    def predict(self, request: dict, headers=None) -> dict:
        instances = request["instances"]
        return {"predictions": self.model.predict(instances).tolist()}

if __name__ == "__main__":
    kserve.ModelServer().start([MyModel("my-model")])
```

## Argo Workflows

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata: {generateName: ml-pipeline-}
spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - {name: train, template: train-step}
          - {name: evaluate, template: eval-step, dependencies: [train]}
          - name: deploy
            template: deploy-step
            dependencies: [evaluate]
            when: "{{tasks.evaluate.outputs.parameters.auc}} > 0.85"
```

```bash
argo submit workflow.yaml
argo list
argo get <workflow-name>
argo logs <workflow-name>
```

## Kubeflow Pipelines SDK (v2)

```python
from kfp import dsl, compiler

@dsl.component
def train(data_path: str) -> str: ...

@dsl.pipeline(name="my-pipeline")
def pipeline(data_path: str):
    t = train(data_path=data_path)

compiler.Compiler().compile(pipeline, "pipeline.yaml")
```

```bash
kfp pipeline upload -p my-pipeline pipeline.yaml
kfp run submit -e my-experiment -r my-run -p my-pipeline
```

## GPU scheduling

```yaml
# Pod requesting a whole GPU (requests == limits required)
resources:
  limits:
    nvidia.com/gpu: 1
tolerations:
  - {key: nvidia.com/gpu, operator: Exists, effect: NoSchedule}
nodeSelector:
  accelerator: nvidia-a10g
```

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml
kubectl describe node <gpu-node> | grep -A5 "Allocatable"    # confirm nvidia.com/gpu shows up
nvidia-smi                                                   # inside a GPU node/pod, check GPU utilization
```

```bash
# eksctl: GPU-tainted node group
eksctl create nodegroup --cluster my-cluster --name gpu-nodes \
  --node-type g5.2xlarge --nodes-min 0 --nodes-max 5 \
  --node-labels accelerator=nvidia-a10g \
  --node-taints nvidia.com/gpu=present:NoSchedule
```
