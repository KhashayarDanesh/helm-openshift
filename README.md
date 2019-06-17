# Helm Openshift

Guide to helm deployment on Openshift Kubernetes Distribution (OKD) in a single namespace without admin role.

You'll need oc (openshift commandline) installed and logged in to your cluster.

in this guide we'll install helm and deploy a project with it.

intro: 

Helm is a package manager for kubernetes which takes care of deployments services etc itself, for example gitlab is a helm deployment, you get 
to install gitlab with helm with a few lines of commands.

Helm itself is a deployment which can be installed both system wide and in a specific project which only your user has access to, so not being an admin is not an issue, 
but how does it work? 

Considering we have it deployed on a project (namespace), and we want to deploy an application on another namespace, tiller's service account will be added to that namespace with edit role, so helm can deploy services, define routes, change configmaps, sercrets etc. 

## `1`. Create an openshift project for tiller or use an already created project

(tiller is the serverside software)

in case of creating:
```
$ oc new-project tiller
```
in case of will to select an existing project:
```
$ oc project {project name}
```

## `2`. do `export TILLER_NAMESPACE=tiller` so the name of tiller's namespace environment variable is set into your shell session. 

## `3`. Install helm client on your computer. 

have a look at [helm releases page](https://github.com/helm/helm/releases) for newer releases. 
please note your helm client and server deployment should be the same version.

Installing using package managers:

On Mac: 
```
$ homebrew install helm

$ helm init --client-only
```

On Ubuntu:
 ```
$ sudo snap install helm --classic

$ helm init --client-only
```

Or alternatively use binary releases: 

On Mac:
```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-darwin-amd64.tar.gz | tar xz
$ cd darwin-amd64
$ ./helm init --client-only
```

On Linux:
```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz | tar xz
$ cd linux-amd64
$ sudo cp helm /usr/local/bin && chmod +x /usr/local/bin/helm 
$ helm init --client only
```

## `4`. Install Helm on the selected namespace

```
$ oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.11.0 | oc create -f -
```
Again, please note helm client and server versions should be the same. 

Then:
```
$ oc rollout status deployment tiller
```
Make sure the deployment is successfully rolled out and then run: 

You should get this : 
```
$ helm version
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
```
remember to set `TILLER_NAMESPACE=tiller` in your environment variables, otherwise you'll experience issues like this: 

the issue is when this is unset, helm client searches for helm server in the wrong namespace (kube-system). 

```
$ helm version
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Error: pods is forbidden: User "KhashayarDanesh" cannot list pods in the namespace "kube-system": User "KhashayarDanesh" cannot list pods in project "kube-system"
```
After resolving all the issues, you can proceed to deploy apps with helm. 

## `5`. Create/Select a project which you want to deploy your apps on.

```
$ oc new-project sample-app
```

## `6`. Grant helm's service account to edit deployment config on this new project: 

```
$ oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
```

## `7`. Grant permissions to run with root access for the containers: 
```
oc adm policy add-scc-to-user anyuid -z default -n sample-app 
```

TaDa ! Now you get to deploy anything you want on the desired namespace using helm. 
