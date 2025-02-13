# gitops-ocp-vm
Managing Virtual Machines using openshift virtualization with Gitops

### Prerequisites 

Operators - Installed from operator hub
 - Openshift Gitops (ArgoCD)
 - Openshift virtualization (Kubevirt)

CLI
 - oc
 - argocd


[Note](https://docs.openshift.com/gitops/1.15/installing_gitops/installing-argocd-gitops-cli.html):   
The Red Hat OpenShift GitOps argocd CLI tool is a Technology Preview feature only. Technology Preview features are not supported with Red Hat production service level agreements (SLAs) and might not be functionally complete. Red Hat does not recommend using them in production. 


Note: Before creating the argocd Application  
Since this example is to deploy virtual machine using openshift virtualization with openshift gitops (ArgoCD)   
Use the role binding to allow permission for openshift-gitops service account.  

Use the command

```bash
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n ${NAMESPACE}
```

or Use the declarative [yaml](Argocd/role-binding.yaml).

```
oc apply -f Argocd/role-binding.yaml
```


In this example [Kustomization](virtual-machine) is used for managing manifests in ArgoCD.



### Steps to create ArgoCD applications to run the virtual machines

__Imperative__

 - Login to ArgoCD CLI

```bash
ADMIN_PASSWD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
```

```bash
SERVER_URL=$(oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.status.ingress[0].host}')
```

```bash
argocd login --username admin --password ${ADMIN_PASSWD} ${SERVER_URL} --insecure --grpc-web
```

 - Create a project (If not required use default project)

```bash
argocd proj create ${ARGOCD_PROJECT}
```

 - Once created, verify the project is in the list

```bash
argocd proj list
```

 - Add cluster destination and the namespace that needs to be allowed for this project

```bash
argocd proj add-destination ${ARGOCD_PROJECT}   https://kubernetes.default.svc '*'
```
 
 - Allow the cluster scoped resources that can be used in this project.

```bash
argocd proj allow-cluster-resource ${ARGOCD_PROJECT} '*' '*'
```

 - Add repository to project 

```bash
argocd repo add ${GIT_REPO} --username ${GIT_USERNAME} --password ${GIT_PASSWORD} --project ${ARGOCD_PROJECT}
```

 - Check that the repository status whether it is connected successfully or not

```bash
argocd repo list
```

 - Create argocd application with automated sync 

```bash
argocd app create ${ARGOCD_APP_NAME} \
  --repo ${GIT_REPO} \
  --path ${KUSTOMIZE_DIR} \
  --dest-namespace ${DEST_NAMESPACE} \
  --dest-server https://kubernetes.default.svc \
  --auto-prune \
  --self-heal \
  --sync-policy automated \
  --sync-option CreateNamespace=false \
  --project ${ARGOCD_PROJECT}
```

 - To view argocd web console.   
 Use the server URL to open the argocd web console in browser.

```bash
oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.status.ingress[0].host}'
```

__Declarative__

To use declarative way of adding the application to argocd. Do the following 

 - Create argocd project [yaml](Argocd\project.yaml)

```
oc apply -f Argocd\project.yaml
```

 - Add git repository to the argocd [yaml](Argocd\secret-git-repo.yaml)

```
oc apply -f Argocd\secret-git-repo.yaml
```

 - Add the argocd Application [yaml](Argocd\dev-rhel9-application.yaml)

```
oc apply -f Argocd\dev-rhel9-application.yaml
```



Refered from:

https://www.redhat.com/en/blog/using-red-hat-advanced-cluster-management-and-openshift-gitops-to-manage-openshift-virtualization


Issues references:

https://github.com/argoproj/argo-cd/issues/15317

https://github.com/kubevirt/kubevirt/issues/11813


