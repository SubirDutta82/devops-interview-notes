```
CrashLoopBackOff — Example Error Outputs

Four example kubectl describe pod <pod> outputs, each showing a different root cause.

1. Image Pull Error
Name:             web-app-7d9f8c6b5-xk2p1
Namespace:        default
Priority:         0
Node:             worker-node-2/172.18.0.5
Start Time:       Thu, 20 Aug 2026 09:12:03 +0000
Labels:           app=web-app
Status:           Pending
IP:               <none>
Containers:
  web-app:
    Container ID:
    Image:          myregistry.io/web-app:v2.4.1-typo
    Image ID:
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9f2xk (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  3m12s                  default-scheduler  Successfully assigned default/web-app-7d9f8c6b5-xk2p1 to worker-node-2
  Normal   Pulling    2m5s (x4 over 3m10s)   kubelet            Pulling image "myregistry.io/web-app:v2.4.1-typo"
  Warning  Failed     2m3s (x4 over 3m8s)    kubelet            Failed to pull image "myregistry.io/web-app:v2.4.1-typo": rpc error: code = NotFound desc = failed to pull and unpack image: myregistry.io/web-app:v2.4.1-typo: not found
  Warning  Failed     2m3s (x4 over 3m8s)    kubelet            Error: ErrImagePull
  Normal   BackOff    118s (x6 over 3m7s)    kubelet            Back-off pulling image "myregistry.io/web-app:v2.4.1-typo"
  Warning  Failed     105s (x6 over 3m7s)    kubelet            Error: ImagePullBackOff

Signal: Image ID: is blank, State: Waiting / Reason: ImagePullBackOff, and Events repeat Failed to pull image → ErrImagePull → ImagePullBackOff. Likely cause: wrong tag/typo in image name, private registry auth missing, or image genuinely doesn't exist.

2. Failed Liveness/Readiness Probe
Name:             api-server-6c7f9d8b4-mz7lq
Namespace:        default
Priority:         0
Node:             worker-node-1/172.18.0.4
Start Time:       Thu, 20 Aug 2026 09:20:11 +0000
Labels:           app=api-server
Status:           Running
IP:               10.42.0.23
Containers:
  api-server:
    Container ID:  containerd://a1b2c3d4e5f6...
    Image:         myregistry.io/api-server:v1.9.0
    Image ID:      docker.io/myregistry.io/api-server@sha256:abc123...
    Port:          8080/TCP
    State:          Running
      Started:      Thu, 20 Aug 2026 09:24:40 +0000
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Thu, 20 Aug 2026 09:22:10 +0000
      Finished:     Thu, 20 Aug 2026 09:24:35 +0000
    Ready:          False
    Restart Count:  5
    Liveness:       http-get http://:8080/healthz delay=5s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:8080/ready delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age                  From     Message
  ----     ------     ----                 ----     -------
  Normal   Pulled     4m ago               kubelet  Container image already present on machine
  Normal   Created    4m ago               kubelet  Created container api-server
  Normal   Started    4m ago               kubelet  Started container api-server
  Warning  Unhealthy  2m30s (x8 over 3m)   kubelet  Liveness probe failed: Get "http://10.42.0.23:8080/healthz": dial tcp 10.42.0.23:8080: connect: connection refused
  Warning  Unhealthy  2m20s (x6 over 2m50s) kubelet Readiness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    2m15s                kubelet  Container api-server failed liveness probe, will be restarted
  Warning  BackOff    1m30s (x3 over 2m)   kubelet  Back-off restarting failed container

Signal: Liveness:/Readiness: lines present, Restart Count climbing, Last State: Terminated / Exit Code: 137, and Events show Unhealthy → Killing → BackOff. Likely cause: app takes longer to start than delay/period allow, wrong probe port/path, or the app itself is unhealthy (crashing internally, slow dependency).

3. Misconfigured Environment Variables
Name:             payment-service-5f8b9c7d6-tq4vn
Namespace:        default
Priority:         0
Node:             worker-node-3/172.18.0.6
Start Time:       Thu, 20 Aug 2026 09:30:02 +0000
Labels:           app=payment-service
Status:           Running
IP:               10.42.0.31
Containers:
  payment-service:
    Container ID:  containerd://f9e8d7c6b5a4...
    Image:         myregistry.io/payment-service:v3.2.0
    Image ID:      docker.io/myregistry.io/payment-service@sha256:def456...
    Port:          9090/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Thu, 20 Aug 2026 09:31:50 +0000
      Finished:     Thu, 20 Aug 2026 09:31:52 +0000
    Ready:          False
    Restart Count:  9
    Environment:
      DB_HOST:      <set to the key 'host' of config map 'payment-db-config'>  Optional: false
      DB_PASSWORD:  <set to the key 'password' in secret 'payment-db-secret'>  Optional: false
      STRIPE_KEY:   <set to the key 'stripe-key' in secret 'payment-secrets'>  Optional: false
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age                   From     Message
  ----     ------     ----                  ----     -------
  Normal   Pulled     3m ago                kubelet  Container image already present on machine
  Normal   Created    3m ago                kubelet  Created container payment-service
  Normal   Started    3m ago                kubelet  Started container payment-service
  Warning  BackOff    45s (x9 over 2m50s)   kubelet  Back-off restarting failed container
$ kubectl logs payment-service-5f8b9c7d6-tq4vn --previous
panic: STRIPE_KEY environment variable is empty, cannot initialize payment client
goroutine 1 [running]:
main.initStripeClient(...)
	/app/payment/stripe.go:42
main.main()
	/app/main.go:28 +0x1a5

Signal: Environment: shows the variable sourced correctly from a ConfigMap/Secret, but kubectl logs --previous reveals the actual value resolved empty — meaning the key exists in the reference but the ConfigMap/Secret key itself has no value (or a wrong key name was referenced upstream). Likely cause: empty/wrong value in the Secret, key name mismatch, or the wrong Secret/ConfigMap version deployed.

4. Volume Mount / Secret / ConfigMap Issue
Name:             worker-job-8d6c5b4a3-vn9pl
Namespace:        default
Priority:         0
Node:             worker-node-1/172.18.0.4
Start Time:       Thu, 20 Aug 2026 09:40:15 +0000
Labels:           app=worker-job
Status:           Pending
IP:               <none>
Containers:
  worker-job:
    Container ID:
    Image:         myregistry.io/worker-job:v1.0.3
    Image ID:
    Port:          <none>
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/config from app-config (rw)
      /etc/secrets from app-secrets (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7lm2q (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  app-config:
    Type:       ConfigMap
    Name:       worker-job-config
    Optional:   false
  app-secrets:
    Type:       Secret
    SecretName: worker-job-secrets
    Optional:   false
Events:
  Type     Reason       Age                 From               Message
  ----     ------       ----                ----               -------
  Normal   Scheduled    2m ago              default-scheduler  Successfully assigned default/worker-job-8d6c5b4a3-vn9pl to worker-node-1
  Warning  FailedMount  38s (x8 over 2m)    kubelet            MountVolume.SetUp failed for volume "app-config" : configmap "worker-job-config" not found
  Warning  FailedMount  20s (x9 over 2m)    kubelet            MountVolume.SetUp failed for volume "app-secrets" : secret "worker-job-secrets" not found

Signal: Pod stuck in Pending / ContainerCreating, Image ID never populates (container never starts), and Events repeatedly show FailedMount ... not found. Likely cause: ConfigMap/Secret not created, created in the wrong namespace, or name mismatch between the Deployment spec and the actual object.

```
```mermaid
flowchart TD
    Start([Pod enters CrashLoopBackOff]) --> Step1

    subgraph Step1 [" 1. Confirm Pod State "]
        A1["kubectl get pods"]
        A2["Check RESTARTS count"]
        A3["Single pod or multiple pods affected?"]
        A1 --> A2 --> A3
    end

    Step1 --> Step2

    subgraph Step2 [" 2. Inspect Logs "]
        B1["kubectl logs pod"]
        B2["Container exits too fast?"]
        B3["kubectl logs --previous pod"]
        B4["Look for: stack traces, missing env vars, dependency failures"]
        B1 --> B2
        B2 -- Yes --> B3 --> B4
        B2 -- No --> B4
    end

    Step2 --> Step3

    subgraph Step3 [" 3. Describe Pod "]
        C1["kubectl describe pod pod"]
        C2{"Check Events for:"}
        C3["Image pull errors"]
        C4["Failed liveness/readiness probes"]
        C5["Misconfigured env vars"]
        C6["Volume mount / secret / configmap issues"]
        C1 --> C2
        C2 --> C3
        C2 --> C4
        C2 --> C5
        C2 --> C6
    end

    Step3 --> Step4

    subgraph Step4 [" 4. Correlate with Deployment "]
        D1["Review recent deployment/statefulset changes"]
        D2["kubectl rollout history deployment name"]
        D3["Check CI/CD pipeline logs"]
        D1 --> D2 --> D3
    end

    Step4 --> Decision{"Root Cause Identified?"}

    Decision -- "Application panic" --> R1["Fix code / adjust startup args"]
    Decision -- "Env misconfigured" --> R2["Correct env vars or secrets"]
    Decision -- "Dependency missing" --> R3["Ensure service/DB reachable"]
    Decision -- "Probe misconfigured" --> R4["Adjust liveness/readiness thresholds"]

    R1 --> Step5
    R2 --> Step5
    R3 --> Step5
    R4 --> Step5

    subgraph Step5 [" 5. Remediate & Redeploy "]
        E1["Apply fix"]
        E2["Redeploy pod"]
        E3["Monitor kubectl get pods -w until stable"]
        E1 --> E2 --> E3
    end

    Step5 --> Step6

    subgraph Step6 [" 6. Document & Prevent "]
        F1["Record incident in runbooks"]
        F2["Add alerts for probe failures & restart counts"]
        F3["Add CI/CD checks for env vars & dependencies"]
        F1 --> F2 --> F3
    end

    Step6 --> End([Pod Stable / Incident Closed])

    classDef startEnd fill:#1b5e20,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef decision fill:#e65100,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef remediate fill:#0d47a1,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef node1 fill:#37474f,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef node2 fill:#4527a0,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef node3 fill:#00695c,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef node4 fill:#6a1b9a,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef node5 fill:#283593,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef node6 fill:#4e342e,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef checkNode fill:#c62828,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold

    class Start,End startEnd
    class Decision decision
    class R1,R2,R3,R4 remediate
    class A1,A2,A3 node1
    class B1,B2,B3,B4 node2
    class C1,C3,C4,C5,C6 node3
    class C2 checkNode
    class D1,D2,D3 node4
    class E1,E2,E3 node5
    class F1,F2,F3 node6

    style Step1 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
    style Step2 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
    style Step3 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
    style Step4 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
    style Step5 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
    style Step6 fill:#102027,stroke:#000000,stroke-width:2px,color:#ffffff
```



```mermaid
flowchart TD
    Start(["describe pod shows a problem"]) --> Check{"Where is the signal?"}

    Check -->|"Image ID blank + Events: ErrImagePull / ImagePullBackOff"| E1["Image Pull Error"]
    Check -->|"Liveness/Readiness lines present + Events: Unhealthy, Killing, BackOff"| E2["Probe Failure"]
    Check -->|"Environment sourced OK but logs show empty/panic on that var"| E3["Env Var Misconfig"]
    Check -->|"Pod stuck Pending/ContainerCreating + Events: FailedMount ... not found"| E4["Volume/Secret/ConfigMap Issue"]

    E1 --> E1C["Cause: wrong image tag/typo,\nmissing registry auth,\nimage doesn't exist"]
    E1C --> E1F["Fix: correct image name/tag,\nadd imagePullSecrets,\nverify image exists in registry"]

    E2 --> E2C["Cause: app slow to start,\nwrong probe port/path,\napp itself unhealthy"]
    E2C --> E2F["Fix: increase initialDelaySeconds,\nfix probe path/port,\nfix underlying app health"]

    E3 --> E3C["Cause: empty/wrong value in\nConfigMap or Secret,\nkey name mismatch"]
    E3C --> E3F["Fix: correct the Secret/ConfigMap\nkey value, redeploy,\nvalidate with kubectl logs --previous"]

    E4 --> E4C["Cause: ConfigMap/Secret not created,\nwrong namespace,\nname mismatch in spec"]
    E4C --> E4F["Fix: create missing object,\nmatch namespace,\nfix name reference in pod spec"]

    E1F --> Redeploy(["Redeploy & Monitor"])
    E2F --> Redeploy
    E3F --> Redeploy
    E4F --> Redeploy

    classDef startEnd fill:#1b5e20,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef decision fill:#e65100,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef errNode fill:#b71c1c,stroke:#000000,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef causeNode fill:#4527a0,stroke:#000000,stroke-width:1px,color:#ffffff
    classDef fixNode fill:#0d47a1,stroke:#000000,stroke-width:1px,color:#ffffff

    class Start,Redeploy startEnd
    class Check decision
    class E1,E2,E3,E4 errNode
    class E1C,E2C,E3C,E4C causeNode
    class E1F,E2F,E3F,E4F fixNode
```
