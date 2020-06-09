# Kubernetes - manage applications with Helm
## What is Helm ?
Helm is for Kubernetes what apt is for Linux.
This is a package manager which allows to easily manage applications deployment in a repeatable manner.
### Helm benefits
* Share deployment manifest 
* Version deployment manifest 
* Split configuration from the deployment code 
* Repeatable / predictable deployment 
* Compose Helm package named Chart to create an application deployment 
### Syntax
The Helm engine is based on go template so that the syntax is composed of Kubernetes yaml manifest and go template in addition to some Helm functions.

Helm uses the Go template engine to separate the deployment configuration from the deployment logic.

**Without Helm**
```
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes
spec:
 type: ClusterIP
 ports:
 - port: 80
   targetPort: 8080
 selector:
  app: hello-kubernetes
  
---

apiVersion: apps/v1
kind: Deployment
metadata:
 name: hello-kubernetes
spec:
 replicas: 2
 selector:
  matchLabels:
   app: hello-kubernetes
 template:
  metadata:
   labels:
    app: hello-kubernetes
  spec:
   containers:
   - name: hello-kubernetes
     image: paulbouwer/hello-kubernetes:1.8
     ports:
     - containerPort: 8080
     env:
     - name: MESSAGE
       value: Hello World
```

With a such Kubernetes manifest manage multiple environment implies to either :

* Create one manifest file per environment
* Introduce some placeholder and create scripts to replace it from configuration files

**With Helm**
```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app }}
spec:
 type: ClusterIP
 ports:
 - port: 80
   targetPort: {{ .Values.port }}
 selector:
  app: {{ .Values.app }}
  
---

apiVersion: apps/v1
kind: Deployment
metadata:
 name: {{ .Values.app }}
spec:
 replicas: "{{ .Values.replicas }}"
 selector:
  matchLabels:
   app: {{ .Values.app }}
 template:
  metadata:
   labels:
    app: {{ .Values.app }}
  spec:
   containers:
   - name: hello-kubernetes
     image: "{{ .Values.image }}:{{ .Values.tag }}"
     ports:
     - containerPort: {{ .Values.port }}
     env:
     - name: MESSAGE
       value: {{ .Values.message}}
```
The configuration code (value.yaml) :
```
app: hello-kubernetes
port: 8080
image: paulbouwer/hello-kubernetes
tag: 1.8
replicas: 2
message: My message coming from the config file
```

### Helm versions
They are two production versions of Helm, the version 2 based on a client server architecture and the version 3 which replaces the server side with a Helm library.

## Hands-on
### Deploy from your own chart
In this use case we will create a simple hello world web site and deploy it with a Helm chart.
### helm create website
Folder structure
* charts folder: Used to contain chart dependencies / subcharts 
* template folder: Contains the template files which are the Kubernetes parametrized manifest 
* values.yaml: Used to configure the chart 
* Chart.yaml: Used to describe the chart 

Read the official documentation Templates folder

Helm creates a package structure allowing us to deploy an application. We can see a serviceaccount.yaml file used to grant your application with permissions on the Kubernetes stack, the ingress.yaml and service.yaml files are used to access the application from outside the stack. Finally the deployment.yaml is used to deploy the application.

By default Helm creates a deployment object which is convenient for most of the use cases, however if you want to use a Kubernetes statefulset you will have to create the associated manifest file into the templates folder.

### Configure the deployment
All the deployment parameters are gathered into the values.yaml file, by default Helm configures a Nginx stack deployment. We will change the configuration to deploy the httpd container. The deployment customization are in the values-apache.yaml file.

The default configuration exposes the port 80, to use another port you have to edit the deployment.yml file (copy the file to the Helm chart folder).

### Install / upgrade
The command below performs an install or update depending of if the application was already deployed or not.

`helm upgrade --install --namespace helm-demo -f website/values.yaml -f website/values-apache.yaml website ./website`

We can customize the message in using a lifecycle hook in the deployment object.
```
lifecycle:
    postStart:
        exec:
        command: ["/bin/sh", "-c", "echo Hello world \
        $(hostname -f) > \
        /usr/local/apache2/htdocs/index.html"]
```

**Delete the release**

The command below is used to delete a Helm release
```
helm delete website
helm list --all
```

**Learning**
* A simple delete does not delete the Helm release history 

Completely remove the release
```
helm delete website --purge
helm list --all
```

Deploy from an existing Chart

Helm uses multiple way to configure the chart to be deployed, you can use multiple values file in addition to command line interface (cli) parameters.

**Install**
```
kubectl create namespace helm-demo2
helm install --namespace helm-demo2 \
   --name website2 -f values.yaml \
   --set cloneHtdocsFromGit.repository="https://lgil3:$GITHUB_PAT@github.dxc.com/Platform-DXC/devcloud-docs.git" \
   bitnami/apache --version 7.3.16
```

The above command will read the configuration from the values.yaml file and overwrite the parameter cloneHtdocsFromGit.repository with the cli value. This precedence mechanism is useful to pass secrets to the chart from the execution context.

**Verify**
Verify pods are up and ready
```
kubectl get pods -n helm-demo2
```
List the installed Helm releases
`helm list`

**Navigate**
Open the http://localhost url to read the devcloud web site.

**Learnings**
* Helm can deploy packages from Helm repositories. 
* A Helm chart can be configured from a configuration file or from the CLI 
* There is a configuration order of precedence between Helm configuration files and CLI 

**Upgrade**
```
helm upgrade --namespace helm-demo2 -f values.yaml \
--set cloneHtdocsFromGit.repository="https://lgil3:$GITHUB_PAT@github.dxc.com/Platform-DXC/devcloud-docs.git" \
website2 bitnami/apache --version 7.3.17
```

**List history version**
```
helm history website2
```

**Rollback**
```
helm rollback website2 1
```

**Advanced usage**

**Helm chart rendering**
```
helm template -f values.yaml -f values-apache.yaml ./
```

**Inject files in template**
* Create a configuration file 
```
echo '#Configuration file' > config
echo 'image: ' >> config
echo 'tag: ' >> config
```
*  Create a configmap file 
```
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:52:05Z
  name: config
data:
  config: |-
    {{ .Files.Get "config" | indent 4 }}
```
  
* Render the chart 
```
helm template -f values.yaml -f values-apache.yaml ./
```

**Templating**
*  Replace the content of the configmap with 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  config: |-
    {{ .Files.Get "config" | indent 4 }}
  config2: |-
    {{ tpl (.Files.Get "config") . | indent 4 }}
```

**Loop**
* Replace the content of the configmap with 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  config: |-
    {{ .Files.Get "config" | indent 4 }}
  config2: |-
    {{ tpl (.Files.Get "config") . | indent 4 }}
  {{- $files := .Files }}
  {{- range tuple "values.yaml" "values-apache.yaml"}}
  {{ . }}: |-
    {{ $files.Get . | indent 4 }}
  {{- end }}
```

**Flow Control**
* Replace the content of the configmap with 

apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
{{ if and $.Values.replicaCount (gt $.Values.replicaCount 1.0) }}
  config: |-
  {{ .Files.Get "config" | indent 4 }}
{{ else }}
  config2: |-
    {{ tpl (.Files.Get "config") . | indent 4 }}
  {{- $files := .Files }}
  {{- range tuple "values.yaml" "values-apache.yaml"}}
  {{ . }}: |-
    {{ $files.Get . | indent 4 }}
  {{- end }}
{{end}}
`
