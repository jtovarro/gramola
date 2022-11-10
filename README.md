# Install ArgoCD and Pipelines using Red Hat Advanced Cluster Management for Kubernetes (ACM)

Red Hat Advanced Cluster Management for Kubernetes provides end-to-end management visibility and control to manage your Kubernetes environment. Take control of your application modernization program with management capabilities for cluster creation, application lifecycle, and provide security and compliance for all of them across data centers and hybrid cloud environments. Clusters and applications are all visible and managed from a single console, with built-in security policies. Run your operations from anywhere that Red Hat OpenShift runs, and manage any Kubernetes cluster in your fleet.

If you have already deployed an instance of ACM or you want to [install](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3/html/install/index) it and leverage the Governance, Risk, and Compliance (GRC) super powers for this demo, use this [policies](rhacm) to get the operators installed automatically in your behalf.

# Install ArgoCD and Pipelines using the operators

Install ArgoCD Operator with OCP OAuth and Openshift Pipelines:

```sh
until oc apply -k util/bootstrap/; do sleep 2; done
```

# Install Gitea

Install Gitea Operator and use it to install Gitea in `gitea-system`:

```sh
until oc apply -k util/gitea/; do sleep 2; done
```

# Migrate from github

Let's prepare some environment variables:

```sh
export GIT_REVISION=main

# Gitea
export GIT_USERNAME=opentlc-mgr
export GIT_PASSWORD=r3dh4t1!

export BASE_REPO_NAME=gramola
export EVENTS_REPO_NAME=gramola-events
export GATEWAY_REPO_NAME=gramola-gateway

# Source Repo to Migrate from
export GIT_USERNAME_SRC=cvicens
export GIT_URL_BASE_SRC=https://github.com/atarazana
export GIT_URL_EVENTS_SRC=${GIT_URL_BASE_SRC}/${EVENTS_REPO_NAME}
export GIT_URL_GATEWAY_SRC=${GIT_URL_BASE_SRC}/${GATEWAY_REPO_NAME}
export GIT_CONF_URL_SRC=${GIT_URL_BASE_SRC}/${BASE_REPO_NAME}
```

We need some sensitive data, your PAT for the source repo:

```sh
echo "Enter PAT for 'https://github.com': " && read -s GIT_PASSWORD_SRC
echo "PAT entered: ${GIT_PASSWORD_SRC}"
```

Get the Gitea host and generate a PAT:

```sh
export GIT_HOST=$(oc get route/repository -n gitea-system -o jsonpath='{.spec.host}')

export GIT_PAT=$(curl -k -s -X 'POST' -H "Content-Type: application/json"  -k -d '{"name":"cicd'"${RANDOM}"'"}' -u ${GIT_USERNAME}:${GIT_PASSWORD} https://${GIT_HOST}/api/v1/users/${GIT_USERNAME}/tokens | jq -r .sha1)

echo "GIT_PAT=${GIT_PAT}"
```

```sh
curl -X 'POST' \
  "https://${GIT_HOST}/api/v1/repos/migrate?token=${GIT_PAT}" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "auth_password": "'${GIT_PASSWORD_SRC}'",
  "auth_username": "'${GIT_USERNAME_SRC}'",
  "clone_addr": "'${GIT_CONF_URL_SRC}'.git",
  "description": "gramola conf",
  "issues": false,
  "labels": false,
  "lfs": false,
  "milestones": false,
  "private": true,
  "pull_requests": false,
  "releases": false,
  "repo_name": "'${BASE_REPO_NAME}'",
  "repo_owner": "'${GIT_USERNAME}'",
  "service": "git",
  "wiki": false
}'

curl -X 'POST' \
  "https://${GIT_HOST}/api/v1/repos/migrate?token=${GIT_PAT}" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "auth_password": "'${GIT_PASSWORD_SRC}'",
  "auth_username": "'${GIT_USERNAME_SRC}'",
  "clone_addr": "'${GIT_URL_EVENTS_SRC}'.git",
  "description": "gramola events",
  "issues": false,
  "labels": false,
  "lfs": false,
  "milestones": false,
  "private": true,
  "pull_requests": false,
  "releases": false,
  "repo_name": "'${EVENTS_REPO_NAME}'",
  "repo_owner": "'${GIT_USERNAME}'",
  "service": "git",
  "wiki": false
}'

curl -X 'POST' \
  "https://${GIT_HOST}/api/v1/repos/migrate?token=${GIT_PAT}" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "auth_password": "'${GIT_PASSWORD_SRC}'",
  "auth_username": "'${GIT_USERNAME_SRC}'",
  "clone_addr": "'${GIT_URL_GATEWAY_SRC}'.git",
  "description": "gramola events",
  "issues": false,
  "labels": false,
  "lfs": false,
  "milestones": false,
  "private": true,
  "pull_requests": false,
  "releases": false,
  "repo_name": "'${GATEWAY_REPO_NAME}'",
  "repo_owner": "'${GIT_USERNAME}'",
  "service": "git",
  "wiki": false
}'
```

# Add plugin section to ArgoCD Custom Resource

We're using a custom plugin called `kustomized-helm` if you're interested have a look at section `configManagementPlugins` in `./util/bootstrap/2.openshift-gitops-patch`.

# Log in ArgoCD with CLI

Log in using the OpenShift SSO using this command:

```sh
./util/argocd-login.sh
```

Alternatively you can log in using this other set of commands:

```sh
export ARGOCD_HOST=$(oc get route/openshift-gitops-server -o jsonpath='{.status.ingress[0].host}' -n openshift-gitops)

export ARGOCD_USERNAME=admin
export ARGOCD_PASSWORD=$(oc get secret openshift-gitops-cluster -o jsonpath='{.data.admin\.password}' -n openshift-gitops | base64 -d)

argocd login $ARGOCD_HOST --insecure --grpc-web --username $ARGOCD_USERNAME --password $ARGOCD_PASSWORD
```

# Register repos

In this guide we cover the case of a protected git repositories that's why you need to create a Personal Access Token so that you don't have to expose your personal account.

You can find an easy guide step by step by following this link: [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

NOTE: We're covering Github in this guide if you use a different git server you may have to do some adjustments.

In order to refer to a repository in ArgoCD you have to register it before, the next command will do this for you asking for the repo url and the the Personal Access Token (PAT) to access to the repository. 


```sh
argocd repo add https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}.git --username ${GIT_USERNAME} --password ${GIT_PAT} --upsert --grpc-web
```

Run this command to list the registered repositories.

```sh
argocd repo list
```

# Register additional clusters

First make sure there is a context with proper credentials, in order to achieve this please log in the additional cluster.


```sh
export API_USER=opentlc-mgr
export API_SERVER_MANAGED=api.example.com:6443
oc login --server=https://${API_SERVER_MANAGED} -u ${API_USER} --insecure-skip-tls-verify
```

Give a name to the additional cluster and add it with the next command.

**CAUTION:** **CLUSTER_NAME** is a name you choose for your cluster, **API_SERVER** is the host and port **without `http(s)`**.

```sh
export CLUSTER_NAME=aws-managed1
CONTEXT_NAME=$(oc config get-contexts -o name | grep ${API_SERVER_MANAGED})

argocd cluster add ${CONTEXT_NAME} --name ${CLUSTER_NAME}
```

Check if your cluster has been added correctly.

```sh
argocd cluster list
```

# Add ArgoCD Project definitions

**IMPORTANT:** Now you can log back in the cluster where ArgoCD is running.

```sh
argocd proj list
```

# Create Root Apps


```sh
cat <<EOF | kubectl apply -n openshift-gitops -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gramola
  namespace: openshift-gitops
  labels:
    argocd-root-app: "true"
spec:
  generators:
  - list:
      elements:
      - env: dev
        desc: "Gramola Dev"
      - env: test
        desc: "Gramola Test"
  template:
    metadata:
      name: gramola-root-app-{{ env }}
      namespace: openshift-gitops
      labels:
        argocd-root-app: "true"
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      destination:
        namespace: openshift-gitops
        name: in-cluster
      project: default
      syncPolicy:
        automated:
          selfHeal: true
      source:
        helm:
          parameters:
            - name: baseRepoUrl
              value: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
            - name: username
              value: "${GIT_USERNAME}"
            - name: baseRepoName
              value: "${BASE_REPO_NAME}"
            - name: gitRevision
              value: "${GIT_REVISION}"
        path: apps/{{ env }}
        repoURL: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
        targetRevision: ${GIT_REVISION}
EOF
```

If an additional cluster has been set up:

```sh
cat <<EOF | kubectl apply -n openshift-gitops -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gramola-cloud
  namespace: openshift-gitops
  labels:
    argocd-root-app: "true"
spec:
  generators:
  - list:
      elements:
      - env: test-cloud
        desc: "Gramola Test"
  template:
    metadata:
      name: gramola-root-app-{{ env }}
      namespace: openshift-gitops
      labels:
        argocd-root-app-cloud: "true"
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      destination:
        namespace: openshift-gitops
        name: in-cluster
      project: default
      syncPolicy:
        automated:
          selfHeal: true
      source:
        helm:
          parameters:
            - name: baseRepoUrl
              value: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
            - name: username
              value: "${GIT_USERNAME}"
            - name: baseRepoName
              value: "${BASE_REPO_NAME}"
            - name: gitRevision
              value: "${GIT_REVISION}"
            - name: destinationName
              value: ${CLUSTER_NAME}
        path: apps/{{ env }}
        repoURL: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
        targetRevision: ${GIT_REVISION}
EOF
```

# Pull permissions

For the main cluster...

```sh
export CONTAINER_REGISTRY_SECRET_NAME=$(yq eval '.containerRegistrySecretName' ./apps/cicd/values.yaml)

if [ -z "${CONTAINER_REGISTRY_USERNAME}" ] && [ -z "${CONTAINER_REGISTRY_PASSWORD}" ]; then
    echo "You should provide a value for CONTAINER_REGISTRY_USERNAME and CONTAINER_REGISTRY_PASSWORD"
else
oc create -n gramola-dev secret docker-registry ${CONTAINER_REGISTRY_SECRET_NAME} \
  --docker-server=https://${CONTAINER_REGISTRY_SERVER} \
  --docker-username=${CONTAINER_REGISTRY_USERNAME} \
  --docker-password=${CONTAINER_REGISTRY_PASSWORD}
oc secrets link default ${CONTAINER_REGISTRY_SECRET_NAME} --for=pull -n gramola-dev
oc create -n gramola-test secret docker-registry ${CONTAINER_REGISTRY_SECRET_NAME} \
  --docker-server=https://${CONTAINER_REGISTRY_SERVER} \
  --docker-username=${CONTAINER_REGISTRY_USERNAME} \
  --docker-password=${CONTAINER_REGISTRY_PASSWORD}
oc secrets link default ${CONTAINER_REGISTRY_SECRET_NAME} --for=pull -n gramola-test
fi
```

If there's an additional cluster...

```sh
export ADDITIONAL_API_SERVER_TOKEN=sha256~Ka7SHU9_Yd4_2OFSIWu1GqM5unovT3PMT8W4h0u7v7Y
export ADDITIONAL_API_SERVER_MANAGED=api.cluster-zmjd7.zmjd7.sandbox1118.opentlc.com:6443
oc login --token=${ADDITIONAL_API_SERVER_TOKEN} --server=https://${ADDITIONAL_API_SERVER_MANAGED}

export CONTAINER_REGISTRY_SECRET_NAME=$(yq eval '.containerRegistrySecretName' ./apps/cicd/values.yaml)

if [ -z "${CONTAINER_REGISTRY_USERNAME}" ] && [ -z "${CONTAINER_REGISTRY_PASSWORD}" ]; then
    echo "You should provide a value for CONTAINER_REGISTRY_USERNAME and CONTAINER_REGISTRY_PASSWORD"
else
oc create -n gramola-test secret docker-registry ${CONTAINER_REGISTRY_SECRET_NAME} \
  --docker-server=https://$CONTAINER_REGISTRY_SERVER \
  --docker-username=$CONTAINER_REGISTRY_USERNAME \
  --docker-password=$CONTAINER_REGISTRY_PASSWORD
oc secrets link default ${CONTAINER_REGISTRY_SECRET_NAME} --for=pull -n gramola-test
fi

oc login -u opentlc-mgr -p r3dh4t1! --server=https://api.cluster-rhpr5.rhpr5.sandbox2409.opentlc.com:6443
```

# Tekton Pipelines

Deploy another ArgoCD app to deploy pipelines.

```sh
cat <<EOF | kubectl apply -n openshift-gitops -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gramola-cicd
  namespace: openshift-gitops
  labels:
    argocd-root-app: "true"
spec:
  generators:
  - list:
      elements:
      - cluster: in-cluster
        ns: openshift-gitops
  template:
    metadata:
      name: gramola-cicd
      namespace: openshift-gitops
      labels:
        argocd-root-app-cloud: "true"
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      destination:
        namespace: '{{ ns }}'
        name: '{{ cluster }}'
      project: default
      syncPolicy:
        automated:
          selfHeal: true
      source:
        helm:
          parameters:
            - name: baseRepoUrl
              value: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
            - name: username
              value: "${GIT_USERNAME}"
            - name: gitRevision
              value: "${GIT_REVISION}"
            - name: containerRegistryServer
              value: ${CONTAINER_REGISTRY_SERVER}
            - name: containerRegistryOrg
              value: ${CONTAINER_REGISTRY_ORG}
        path: apps/cicd
        repoURL: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
        targetRevision: ${GIT_REVISION}
EOF
```

We are going to create secrets instead of storing then in the git repo, but before we do let's check that ArgoCD has created the namespace for us.

```sh
export CICD_NAMESPACE=$(yq eval '.cicdNamespace' ./apps/cicd/values.yaml)
```

NOTE: If the namespace is not there yet, you can check the sync status of the ArgoCD application with: `argocd app sync gramola-cicd-app`

```sh
oc get project ${CICD_NAMESPACE}
```

Once the namespace is created you can create the secrets. This commands will ask you for the PAT again, this time to create a secret with it.

```sh
./apps/cicd/create-git-secret.sh ${CICD_NAMESPACE} ${GIT_HOST} ${GIT_USERNAME} ${GIT_PAT}
```

```sh
export CONTAINER_REGISTRY_SERVER=$(oc get route/myregistry-quay -n quay-system -o jsonpath='{.spec.host}')
export CONTAINER_REGISTRY_ORG=opentlc-mgr
export CONTAINER_REGISTRY_USERNAME="opentlc-mgr+cicd"
```

```sh
echo "Enter password for ${CONTAINER_REGISTRY_USERNAME}: " && read -s CONTAINER_REGISTRY_PASSWORD
echo "Password entered: ${CONTAINER_REGISTRY_PASSWORD}"
```

Create repositories for gramola-event and gramola-gateway and add them to the robot account!

```sh
./apps/cicd/create-registry-secret.sh ${CICD_NAMESPACE} ${CONTAINER_REGISTRY_SERVER} ${CONTAINER_REGISTRY_ORG} ${CONTAINER_REGISTRY_USERNAME} ${CONTAINER_REGISTRY_PASSWORD}
```

## Creating Web Hooks

```sh
EVENTS_CI_EL_LISTENER_HOST=$(oc get route/el-events-ci-pl-push-gitea-listener -n ${CICD_NAMESPACE} -o jsonpath='{.spec.host}')

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/${EVENTS_REPO_NAME}/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "http://'"${EVENTS_CI_EL_LISTENER_HOST}"'"
  },
  "events": [
    "push" 
  ],
  "type": "gitea"
}'

GATEWAY_CI_EL_LISTENER_HOST=$(oc get route/el-gateway-ci-pl-push-gitea-listener -n ${CICD_NAMESPACE} -o jsonpath='{.spec.host}')

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/${GATEWAY_REPO_NAME}/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "http://'"${GATEWAY_CI_EL_LISTENER_HOST}"'"
  },
  "events": [
    "push" 
  ],
  "type": "gitea"
}'
```

```sh
EVENTS_CD_EL_LISTENER_HOST=$(oc get route/el-events-cd-pl-pr-gitea-listener  -n ${CICD_NAMESPACE} -o jsonpath='{.spec.host}')

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/${BASE_REPO_NAME}/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "http://'"${EVENTS_CD_EL_LISTENER_HOST}"'"
  },
  "events": [
    "pull_request" 
  ],
  "type": "gitea"
}'

GATEWAY_CD_EL_LISTENER_HOST=$(oc get route/el-gateway-cd-pl-pr-gitea-listener  -n ${CICD_NAMESPACE} -o jsonpath='{.spec.host}')

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/${BASE_REPO_NAME}/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "http://'"${GATEWAY_CD_EL_LISTENER_HOST}"'"
  },
  "events": [
    "pull_request" 
  ],
  "type": "gitea"
}'
```

# Jenkins Pipelines

Deploy another ArgoCD app to deploy jenkins pipelines.

```sh
cat <<EOF | oc apply -n openshift-gitops -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gramola-cicd-jenkins
  namespace: openshift-gitops
  labels:
    argocd-root-app: "true"
spec:
  generators:
  - list:
      elements:
      - cluster: in-cluster
        ns: openshift-gitops
  template:
    metadata:
      name: gramola-cicd-jenkins
      namespace: openshift-gitops
      labels:
        argocd-root-app-cloud: "true"
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      destination:
        namespace: '{{ ns }}'
        name: '{{ cluster }}'
      project: default
      syncPolicy:
        automated:
          selfHeal: true
      source:
        helm:
          parameters:
            - name: gitUrl
              value: "https://${GIT_HOST}"
            - name: gitUsername
              value: "${GIT_USERNAME}"
            - name: baseRepoName
              value: "${BASE_REPO_NAME}"
            - name: gitRevision
              value: "${GIT_REVISION}"
            #- name: destinationName
            #  value: ${CLUSTER_NAME}
            - name: containerRegistryServer
              value: ${CONTAINER_REGISTRY_SERVER}
            - name: containerRegistryOrg
              value: ${CONTAINER_REGISTRY_ORG}
            - name: proxyEnabled
              value: 'false'
            - name: pipelineClusterName
              value: ''
            - name: pipelineCredentials
              value: ''
        path: apps/cicd-jenkins
        repoURL: "https://${GIT_HOST}/${GIT_USERNAME}/${BASE_REPO_NAME}"
        targetRevision: ${GIT_REVISION}
EOF
```

We are going to create secrets instead of storing then in the git repo, but before we do let's check that ArgoCD has created the namespace for us.

```sh
export JENKINS_NAMESPACE=$(yq eval '.jenkinsNamespace' ./apps/cicd-jenkins/values.yaml)
```

NOTE: If the namespace is not there yet, you can check the sync status of the ArgoCD application with: `argocd app sync gramola-cicd-jenkins`

```sh
oc get project ${JENKINS_NAMESPACE}
```

Once the namespace is created you can create the secrets. This commands will ask you for the PAT again, this time to create a secret with it.

```sh
./apps/cicd-jenkins/create-git-secret.sh ${JENKINS_NAMESPACE} ${GIT_HOST} ${GIT_USERNAME} ${GIT_PAT}
```
<!-- 
```sh
export CONTAINER_REGISTRY_SERVER=$(yq eval '.containerRegistryServer' ./apps/cicd-jenkins/values.yaml)
export CONTAINER_REGISTRY_ORG=$(yq eval '.containerRegistryOrg' ./apps/cicd-jenkins/values.yaml)
export CONTAINER_REGISTRY_USERNAME="varadero+cicd"
```

```sh
echo "Enter password for ${CONTAINER_REGISTRY_USERNAME}: " && read -s CONTAINER_REGISTRY_PASSWORD
echo "Password entered: ${CONTAINER_REGISTRY_PASSWORD}"
``` -->

```sh
./apps/cicd-jenkins/create-registry-secret.sh ${JENKINS_NAMESPACE} ${CONTAINER_REGISTRY_SERVER} ${CONTAINER_REGISTRY_ORG} ${CONTAINER_REGISTRY_USERNAME} ${CONTAINER_REGISTRY_PASSWORD}
```

## Creating Web Hooks

OJO PATCH CONFIGMAP gitea-system/repository antes eliminar owner ref:

    [webhook]

    ALLOWED_HOST_LIST = *

    SKIP_TLS_VERIFY = true


```sh
oc set triggers bc/gramola-events-pipeline --from-webhook -n ${JENKINS_NAMESPACE}
export GIT_WEBHOOK_SECRET_EVENTS=$(oc get bc/gramola-events-pipeline -o jsonpath={.spec.triggers[0].generic.secret} -n ${JENKINS_NAMESPACE})

oc set triggers bc/gramola-gateway-pipeline --from-webhook -n ${JENKINS_NAMESPACE}
export GIT_WEBHOOK_SECRET_GATEWAY=$(oc get bc/gramola-gateway-pipeline -o jsonpath={.spec.triggers[0].generic.secret} -n ${JENKINS_NAMESPACE})
```

```sh
export PATTERN='at (.*)$'
export CURRENT_CONTEXT=$(oc cluster-info | sed -e 's/\x1b\[[0-9;]*m//g' | head -n 1)
[[ ${CURRENT_CONTEXT} =~ ${PATTERN} ]] 
echo "${BASH_REMATCH[0]}"
echo "${BASH_REMATCH[1]}"
export API_SERVER=${BASH_REMATCH[1]}

export EVENTS_CI_BC_WEBHOOK_URL="${API_SERVER}/apis/build.openshift.io/v1/namespaces/${JENKINS_NAMESPACE}/buildconfigs/gramola-events-pipeline/webhooks/${GIT_WEBHOOK_SECRET_EVENTS}/generic"

export GATEWAY_CI_BC_WEBHOOK_URL="${API_SERVER}/apis/build.openshift.io/v1/namespaces/${JENKINS_NAMESPACE}/buildconfigs/gramola-gateway-pipeline/webhooks/${GIT_WEBHOOK_SECRET_GRAMOLA}/generic"

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/gramola-events/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "'"${EVENTS_CI_BC_WEBHOOK_URL}"'"
  },
  "events": [
    "push" 
  ],
  "type": "gitea"
}'

curl -k -X 'POST' "https://${GIT_HOST}/api/v1/repos/${GIT_USERNAME}/gramola-gateway/hooks" \
  -H "accept: application/json" \
  -H "Authorization: token ${GIT_PAT}" \
  -H "Content-Type: application/json" \
  -d '{
  "active": true,
  "branch_filter": "*",
  "config": {
     "content_type": "json",
     "url": "'"${GATEWAY_CI_BC_WEBHOOK_URL}"'"
  },
  "events": [
    "push" 
  ],
  "type": "gitea"
}'
```

# Useful commands

# Sync Root Apps alone

```sh
argocd app sync gramola-root-app-dev
argocd app sync gramola-root-app-test
argocd app sync gramola-root-app-test-cloud
```

# Sync apps manually

```sh
argocd app sync economiacircular-app-dev
argocd app sync economiacircular-app-test
argocd app sync economiacircular-app-test-cloud
```

# Sync children apps (app of apps)

```sh
argocd app sync -l app.kubernetes.io/instance=gramola-root-app
argocd app sync -l app.kubernetes.io/instance=gramola-root-app-dev
argocd app sync -l app.kubernetes.io/instance=gramola-root-app-test
argocd app sync -l app.kubernetes.io/instance=gramola-root-app-test-cloud
```

# AUX
git clone https://oauth2:1AbCDeF_g2HIJKLMNOPqr@gitlab.com/yourusername/project.git project
