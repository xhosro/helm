# Helm Chart Guide

This guide provides step-by-step instructions for using Helm to manage Kubernetes applications. It covers adding Helm repositories, creating and managing Helm charts, handling dependencies, managing secrets, and utilizing Helm hooks.

## Table of Contents

1. [Adding Stable Helm Repository](#adding-stable-helm-repository)
2. [Creating Helm Charts](#creating-helm-charts)
3. [Helm Install, Upgrade, Rollback](#helm-install-upgrade-rollback)
4. [Helm Dependencies](#helm-dependencies)
5. [Creating a Private Helm Chart Repository with S3](#creating-a-private-helm-chart-repository-with-s3)
6. [Helm Secret Management](#helm-secret-management)
7. [Helm Chart Hooks](#helm-chart-hooks)

---

## 1. Adding Stable Helm Repository

Add the stable Helm repository and search for charts:

```bash
helm repo add stable https://charts.helm.sh/stable
helm search repo
```



## 2. Creating Helm Charts

```bash
helm create helloworld
kubectl create namespace dev
helm install -f helloworld/values.yaml -n dev helloworld ./helloworld 
````
         AME: helloworld
         LAST DEPLOYED: Sat Sep 21 17:34:19 2024
         NAMESPACE: dev
        STATUS: deployed
        REVISION: 1
        NOTES:
        1. Get the application URL by running these commands:
        export POD_NAME=$(kubectl get pods --namespace dev -l "app.kubernetes.io/name=helloworld,app.kubernetes.io/instance=helloworld" -o jsonpath="{.items[0].metadata.name}")
        export CONTAINER_PORT=$(kubectl get pod --namespace dev $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
        echo "Visit http://127.0.0.1:8080 to use your application"
        kubectl --namespace dev port-forward $POD_NAME 8080:$CONTAINER_PORT

list helm charts in dev namespace: 

```bash
  helm ls -n dev
```


## 3. Helm Install, Upgrade, Rollback

```bash
helm create app 
helm install --dry-run --set image.tag=1.19.3 -n default app ./app
helm install --set image.tag=1.19.3 -n default app ./app
helm  ls -n default
````
which version of the app we use: 

```bash
kubectl get pods -n default -o jsonpath='{.items[*].spec.containers[*].image}'
````

There are few option when you want to upgrade a helm chart:
  
 1. --atomic
 2. --cleanup-on-fail
 3. --force

upgrade helm chart to new app version:

```bash
helm upgrade --set image.tag=1.19.4 -n default app ./app
```  

rollback to the previous version of app:

```bash
   helm rollback app
```
    
try to upgrade to the wrong version with one minute of waiting and then terminate the pod:

```bash
  helm upgrade --set image.tag=wrong --timeout=1m --atomic --cleanup-on-fail -n default app ./app
```

## 4. Helm Dependencies

for example you need to deploy frontend application & required to database is presented
 there are 3 ways how you can install dependencies in helm chart:

    1.dependencies condition
    2.enabled flags
    3.tags

```bash
    helm create app
    helm create database
```
## 4.1. dependencies condition
    1. add dependencies to chart.yaml file of app helm chart: 

        dependencies:
          - name: database
            version: 0.1.0
            repository: file://../database
            condition: database.enabled

    2. add this to values.yaml

            database: 
              enabled: true
    3. cd app
       helm depandency update
       helm install -f values.yaml web .

       so we have two databases: 

            web-app-559b8cc6ff-tlxkm        1/1     Running
           0             98s
            web-database-5ff88b8f99-jdr7z   1/1     Running

        
    helm del web

## 2. enabled flag

          dependencies:
          - name: database
            version: 0.1.0
            repository: file://../database
            enabled: true


     helm dependency update
     helm install -f values.yaml web .

## 3. tag

                    dependencies:
                    - name: database
                      version: 0.1.0
                      repository: file://../database
                      tags:
                      - database



              add  to values.yaml 

              tags:
                database: true


## 5. Creating a Private Helm Chart Repository with S3

```bash
helm plugin install https://github.com/hypnoglow/helm-s3.git

helm plugin list
````
then we create S3 bucket in AWS
```bash
helm s3 version --mode

helm s3 init s3://<bucket_name>/charts : Initialized empty repository at s3://helm-s3-khosro/charts
```
we can see there is folder chart and inside it index.yaml

give it a name : 
```bash
helm repo add private s3://<bucket_name>/charts

helm repo list

helm create hello-world
```
for upload we need to archive it : 
```bash
helm package hello-world

helm s3 push --relative ./hello-world-0.1.0.tgz private

helm search repo private

helm install -n dev hello-world private/hello-world --version 0.1.0

helm ls -n dev
```

## 6. Helm Secret Management

install secret plugin: 
```bash
helm plugin install https://github.com/zendesk/helm-secrets
```
```bash
helm secrets help
brew install gnu-getopt
```
we use gpg ( GNU privacy guard) to create key pairs, or we can use AWS KMS also
```bash
brew install gpg
gpg --list-keys
gpg --gen-key 
```
so we genrate key pairs.

```
pub   ed25519 2024-09-21 [SC] [expire : 2027-09-21]
      611C4E38A6F928516EA11A43CE08F815E56CF46D #fingerprint
uid                     username <mail@gmail.com>
sub   cv25519 2024-09-21 [E] [expire : 2027-09-21]
```

we installed by default mozilla/sops for creating the secrets: 
```
sops -p <fingerprint> secrets.yaml
```
we delete all lines in vim ( :%d ) then add: password:secret1234
```
helm secrets view secrets.yaml
```
if we encouner the error that can't decrypt : 
```
GPG_TTY=$(tty)
export GPG_TTY
```

and again: 
```bash
helm secrets view secrets.yaml
```
we have a file : secrets.yaml
 
then we create a secrets.yaml in templates repo of app chart

```
apiVersion: v1
kind: Secret
metadata: 
   name: credentials
   labels: 
     app: app
     chart: '{{ .Chart.Name}}-{{ .Chart.Version }}'
     release: '{{ .Release.Name }}'
     heritage: '{{ .Release.Service }}'
type: Opaque
data:
   password: {{ .Values.password | b64enc | quote }}
```

and add env to deployment.yaml

```
   env:
          - name: MY_PASSWORD
            valueFrom:
              secretKeyRef:
                name: credentials
                key: password
```
finally install helm chart: 
```bash
helm secrets install app ./app -n default -f ./secrets.yaml 
helm uninstall app -n default
helm secrets upgrade app ./app -n default -f ./secrets.yaml
helm ls
kubectl get secret credentials -o yaml
echo "YXplcnR5MTIzNA==" | base64 -d
kubectl get pods
kubectl exec app-584cfc49fd-rkj75 -- printenv
```

## Helm Chart Hooks

Allows you to add custom behavior to your Helm charts during the release lifecycle. Helm hooks enable you to define specific actions (such as creating or deleting resources) at particular points during the deployment, upgrade, or deletion of a Helm chart.

```
helm create foo
```
create hooks folder in template repo

create pre-install.yaml and post-install.yaml in folder

```
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "foo.fullname" . }}-pre-install-job-hook"
  annotations:
    "helm.sh/hook": pre-install # post-install 
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: pre-install
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ['sh', '-c', 'echo pre-install Pod is Running ; sleep 10']
      restartPolicy: OnFailure
      terminationGracePeriodSeconds: 0
  backoffLimit: 3
  completions: 1
  parallelism: 1
```

```
helm install test ./foo

watch -n 1 kubectl get pods
helm delete test 
```

pre-install: Executed before the chart resources are installed.

post-install: Executed after the chart resources are installed.

pre-upgrade: Executed before a chart is upgraded.

post-upgrade: Executed after a chart upgrade is completed.

pre-delete: Executed before the chart resources are deleted.

post-delete: Executed after the chart resources are deleted.

pre-rollback: Executed before a rollback happens.

post-rollback: Executed after a rollback.