# Custom metrics for gpu utilization on EKS
> gpu utilization을 보고 hpa(horizontal pod autoscaling)을 하기 위해서

1. [nvidia gpu monitoring](https://nvidia.github.io/gpu-monitoring-tools/)
   1. 기본은 노드를 보는 용도임, pod단위가 아님...
   2. 문제는 namespace가 pod에 있는게 아니라, 이 monitoring 하는 agent의 namespace가 올라옴..
2. kube-prometheus-prometheus:9090 모니터링
3. 실제로 custom metric으로 쓰기 위해선
4. 이걸 prometheus에 있는걸 k8s custom metrics에 붙이기 위해선
   1. [prometheus adapter](https://hub.helm.sh/charts/stable/prometheus-adapter/v0.2.3) 가 필요
   2. prometheus url만 셋팅하면 바로 사용가능
5. custom metric을 뽑을수 있음
```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/monitoring/pods/*/dcgm_gpu_utilization" | jq
```
6. hpa가 namespace안의 리소스를 기준으로 하는듯?? (안찾아봄).. 근데 namespace안의 gpu는 빈 값으로 옮

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/gpu-prod/pods/*/dcgm_gpu_utilization" | jq

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/gpu-prod/pods/*/cpu_user" | jq
```

7. 그래서 pod단위로 주는 prometheus resource가 필요

   1. https://github.com/NVIDIA/gpu-monitoring-tools/tree/master/exporters/prometheus-dcgm/k8s/pod-gpu-metrics-exporter

   2. pre reqired에..

      1. NVIDIA Tesla drivers = R384+ (download from [NVIDIA Driver Downloads page](http://www.nvidia.com/drivers))
      2. nvidia-docker version > 2.0 (see how to [install](https://github.com/NVIDIA/nvidia-docker) and it's [prerequisites](https://github.com/nvidia/nvidia-docker/wiki/Installation-(version-2.0)#prerequisites))
      3. Set the [default runtime](https://github.com/NVIDIA/nvidia-container-runtime#daemon-configuration-file) to nvidia
      4. Kubernetes version = 1.13
      5. Set KubeletPodResources in /etc/default/kubelet: KUBELET_EXTRA_ARGS=--feature-gates=KubeletPodResources=true

   3. Kubelet 실행시 feature-gates 옵션을 추가로 줘야함...

   4. eksctl에서는 현재 지원안해서 issue 날렸더니 수정해서 올려줌.

      1. 아직 테스트 안해봄….
      2. 대략 아래처럼 ..

      ```yaml
        - name: ng-gpu-xxxxxx
          instanceType: p2.xlarge
          minSize: 1
          maxSize: 50
          labels:
            role: gpu-engine
            hardware-type: NVIDIAGPU
          iam:
            withAddonPolicies:
              autoScaler: true
          taints:
            nvidia.com/gpu: "yes:NoSchedule"
          ssh:
            publicKeyName: eks-main
          kubeletExtraConfig:
            featureGates:
              KubeletPodResources: true
              # has to be enabled, otherwise it will be disabled
              RotateKubeletServerCertificate: true
      ```

      3. eksctl에서 kubelet 시작시 config 셋팅 https://github.com/weaveworks/eksctl/blob/ae4cdff698b1e7dbe2cfd7db12ede1d011508f7b/pkg/nodebootstrap/assets/10-eksclt.al2.conf

      ```bash
      $ cat /etc/eksctl/kubelet.yaml
      kind: KubeletConfiguration
      address: 0.0.0.0
      apiVersion: kubelet.config.k8s.io/v1beta1
      ...
      featureGates:
        KubeletPodResources: true
        RotateKubeletServerCertificate: true
      ```

      4. 결과는

      ```json
      $ curl localhost:9400/gpu/metrics
      ...
      dcgm_mem_copy_utilization{gpu="0",uuid="GPU-56e1677a-1440-7298-c32e-9ab8c83e8f55",pod_name="gpuapp-api-74d4c4db77-hwffr",pod_namespace="gpu-prod",container_name="gpuapp-api"} 0
      ...
      ```


