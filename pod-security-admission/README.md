# Objective
- Easier prvilege escalation for an attacker
- disable securityContext (RunAsUser=0  / allowPrivilegeEscalation=true)       




# Pod Security Admission
#Enforce Pod Security Standards by Configuring the Built-in Admission Controller

ref. https://v1-26.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/


1. create AdmissionConfiguration in podsecurity.yaml file
```
apiVersion: apiserver.config.k8s.io/v1 # see compatibility note
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    # Defaults applied when a mode label is not set.
    #
    # Level label values must be one of:
    # - "privileged" (default)
    # - "baseline"
    # - "restricted"
    #
    # Version label values must be one of:
    # - "latest" (default) 
    # - specific version like "v1.26"
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # Array of authenticated usernames to exempt.
      usernames: []
      # Array of runtime class names to exempt.
      runtimeClasses: []
      # Array of namespaces to exempt.
      namespaces: []
```


2. The above manifest needs to be specified via the --admission-control-config-file to kube-apiserver.

#create k3d cluster for only test(do not use this config in Production)
```
k3d cluster create simple --image rancher/k3s:v1.26.4-k3s1 --volume "${PWD}/podsecurity.yaml:/etc/podsecurity.yaml"  --k3s-arg "--kube-apiserver-arg=admission-control-config-file=/etc/podsecurity.yaml@server:*"
```

#check k3d process 
```
docker exec -ti 44063d973fa0 sh
/ # ps -ef
PID   USER     COMMAND
    1 0        /sbin/docker-init -- /bin/k3s server --kube-apiserver-arg=admission-control-config-file=/etc/podsecurity.yaml --tls-san 0.0.0.0
```

ref. https://github.com/shanaider/k3d


3. create ns my-baseline-namespace and enable policy "privileged", "baseline" , "restricted" ,so please select one.

ref. https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/
ref. https://v1-26.docs.kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/#adopt-a-multi-mode-strategy


#Adopt a multi-mode strategy

Blocks any pods that don't satisfy the baseline policy requirements.
Generates a user-facing warning and adds an audit annotation to any created pod that does not meet the restricted policy requirements.
```
k create ns my-baseline-namespace

kubectl label --overwrite ns my-baseline-namespace \
pod-security.kubernetes.io/enforce=baseline \
pod-security.kubernetes.io/enforce-version=v1.27 \
pod-security.kubernetes.io/warn=restricted \
pod-security.kubernetes.io/warn-version=v1.27 \
pod-security.kubernetes.io/audit=restricted \
pod-security.kubernetes.io/audit-version=v1.27

```

4. test deploy with deny policy
ref . https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      privileged: true
      allowPrivilegeEscalation: true
```

```
k create -f pod-deny.yaml
k create -f pod-allow.yaml
```

## Example case

#Because "baseline" profile do not allow securityContext.privileged = true
```
k create -f pod-deny.yaml

Error from server (Forbidden): error when creating "pod-deny.yaml": pods "security-context-demo" is forbidden: violates PodSecurity "baseline:latest": privileged (container "sec-ctx-demo" must not set securityContext.privileged=true)
```


#๋just warning on "restricted" profile but are otherwise allowed.
```
k create -f pod-allow.yaml 

Warning: would violate PodSecurity "restricted:v1.27": allowPrivilegeEscalation != false (container "sec-ctx-demo" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "sec-ctx-demo" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "sec-ctx-demo" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "sec-ctx-demo" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/security-context-demo created
```



#You can check standards prfiles(Privileged / Baseline / Restricted) 
ref. https://kubernetes.io/docs/concepts/security/pod-security-standards/


# Example dockerfile run as non-root
``` 
FROM openjdk:18-alpine3.15 
#Run non-root 
RUN addgroup -S appgroup && adduser -S appuser -G appgroup -h /home/appuser
\
# Set the working directory inside the container 
WORKDIR /app

# set ownership and permissions
RUN Chown -R appuser:appgroup /app 

# switch to user
USER appuser 

# Copy the JAR file to the container
COPY --chown=appuser:appgroup ./target/demo-1.0-SNAPSHOT.jar .

# Set the entrypoint command to run the JAR file
CMD [""java"", ""-jar"", ""/app/demo-1.0-SNAPSHOT.jar""]
```