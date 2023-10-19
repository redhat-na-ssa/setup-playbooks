# Red Hat Developer Hub (Early Access Program) Setup instructions

> :exclamation: NOTE: *the following instructions outlined here are not intended for building a production-ready environment! It is intended to build a PoC leveraging Red Hat Openshift DevSecOps offerings.* 
> This is a live document, so make sure you refresh this page to see the latest version!

## Openshift Cluster Checklist

1. Sandbox **non-production** cluster
2. minimum cluster size recommended _for a PoC_:
 * 3x Control Plane nodes with 
     * 16 vCPUs
     * 64Gb RAM
 * 3x Worker nodes with
     * 32 vCPUs
     * 128Gb RAM
3. Container Registry
 * Red Hat Quay `<-- recommended`
     * ❗**Requires an S3 compatible backend storage class configured in the cluster**
     * eg: Red Hat ODF
 * Openshift Internal Container Registry
  
> :white_check_mark: NOTE: RHDH includes support for these two options OOB

## Openshift DevSecOps Layered components 

> NOTE: the components listed below _may be required_ to enable some of the capabilities available in the Developer Hub instance.
> They are not required to install DevHub, but when available, they enable additional capabilities in the product.

 * Openshift GitOps (ArgoCD) `required for CD`
 * Openshift Pipelines (Tekton) `option for CI`
 * Openshift DevSpaces `Containerized Developer Workspaces (web-based VSCode IDE)`
 * Openshift Advanced Cluster Management (ACM) `required for clusters/topology view`
 * Openshift Advanced Cluster Security (ACS) `required for image security/vulnerability scanning`
 * Red Hat SSO (Keycloak) `required as an iDP broker using OIDC standard`
 * Openshift Quay `required for image registry`

   > All of the components listed above are installed using Operators available in the Openshift Operator Hub

## External 3rd party Integrations
* Source Control System
  * (Enterprise/hosted) GitHub `requires additional setup`
  * (Enterprise/hosted) Gitlab `requires additional setup`
* Continuous Integration
  * Github Actions
  * Gitlab CI
* Identity Provider
  * ❗**Requires OIDC support**
  *  Red Hat SSO (based on Keycloak) is recommended as an Access Manager and iDP broker

---

> ❗❗❗ NOTE: we assume you have access to a Unix-based shell to execute the commands thought the next sections. 
> If you don't have one and is using Windows you can use a Git shell (eg: https://gitforwindows.org/).

> We will be collectiong and creating a couple of environment variables, 
> it may be preferable create a `./env.sh` file containing them and the later run `source ./env.sh` to make them available for the commands we'll be executing.

---

# Openshift Cluster Info

Log in to your OpenShift cluster via the `oc` client.  Set the `OPENSHIFT_CLUSTER_INFO` variable for use later.

``` sh
export OPENSHIFT_CLUSTER_INFO=$(oc cluster-info | head -n 1 | sed 's/^.*https...api//' | sed 's/.6443.*$//')
export K8S_CLUSTER_API=$(oc cluster-info | head -n 1 |  sed 's/^.*https/https/')
```

# Source Control System and Identity Provider setup
> ❗This is the most important integration required by Developer Hub!

## Create GitHub Organization (if not exist)

Create a new [Github Organization](https://github.com/account/organizations/new?plan=free). This organization will contain the code repositories for the `components` created by Red Hat Developer Hub.

The `GITHUB_ORGANIZATION` environment variable will be set to the name of the organization.

> ❗tip
  You may also use any organization you are a member of, as long as you have the ability to create new repositories within it.

``` sh
# if using a hosted Enterprise GitHub replace github.com by your internal domain.
export GITHUB_HOST_DOMAIN=github.com
export GITHUB_ORGANIZATION='Your Github Org Name Here'
export GITHUB_ORG_URL=https://$GITHUB_HOST_DOMAIN/$GITHUB_ORGANIZATION
```

## Set Up GitHub Application

1. Create a new GitHub Application to use the `Git WebHooks` functionality in this demo.  The required field will be populated, and correct permissions set.

    ``` sh
    open "https://$GITHUB_HOST_DOMAIN/organizations/$GITHUB_ORGANIZATION/settings/apps/new?name=$GITHUB_ORGANIZATION-rhdh-app&url=https://janus-idp.io/blog&webhook_active=true&public=false&callback_url=https://developer-hub-rhdh.apps$OPENSHIFT_CLUSTER_INFO/api/auth/github/handler/frame&webhook_url=https://developer-hub-rhdh.apps$OPENSHIFT_CLUSTER_INFO&administration=write&checks=write&actions=write&contents=write&statuses=write&vulnerability_alerts=write&dependabot_secrets=write&deployments=write&discussions=write&environments=write&issues=write&packages=write&pages=write&pull_requests=write&repository_hooks=write&repository_projects=write&secret_scanning_alerts=write&secrets=write&security_events=write&workflows=write&webhooks=write&members=read"
    ```

 > you can also use `echo` instead od `open` to print out the URL, then copy&paste into your web browser address bar.

 * Remember to fill out the following fields:
   * Callback URL: `https://developer-hub-rhdh.apps.your-cluster-domain/api/auth/github/handler/frame`
   * Webhook URL: `https://developer-hub-rhdh.apps.your-cluster-domain/`
   * Webhook Secret: `a radom string` (save it to use later in the DevHub config!)

2. Set the `GITHUB_APP_ID` and `GITHUB_APP_CLIENT_ID` environment variables to the App ID  and App Client ID, respectively. Generate a new client secret and set the `GITHUB_APP_CLIENT_SECRET` environment variable.  Then, generate a `Private Key` for this app and **download** the private key file.
    ``` sh
    export GITHUB_APP_ID=
    export GITHUB_APP_CLIENT_ID=
    export GITHUB_APP_CLIENT_SECRET=
    export GITHUB_APP_PRIVATE_KEY="$(< /path/to/github-app-key.pem)"
    ```

    ![Organization Client Info](assets/org-client-info.png)

3. Go to the `Install App` table on the left side of the page and install the GitHub App that you created for your organization. Choose to install it to All Repositories under this Organization.

    ![Install App](assets/org-install-app.png)

4. source your `env.sh` file
At this point your `env.sh` should looke like this

``` sh
export OPENSHIFT_CLUSTER_INFO=$(oc cluster-info | head -n 1 | sed 's/^.*https...api//' | sed 's/.6443.*$//')
export K8S_CLUSTER_API=$(oc cluster-info | head -n 1 |  sed 's/^.*https/https/')

export GITHUB_HOST_DOMAIN=github.com
export GITHUB_ORGANIZATION='Your Github Org Name Here'
export GITHUB_ORG_URL=https://$GITHUB_HOST_DOMAIN/$GITHUB_ORGANIZATION

export GITHUB_APP_ID='123'
export GITHUB_APP_CLIENT_ID='random string'
export GITHUB_APP_CLIENT_SECRET='random string'
export GITHUB_APP_PRIVATE_KEY="$(< github-app-key.pem)"
```

Source it before going to the next session
``` sh
source env.sh
```
---

# Red Hat Developer Hub
## Installation
### Quay.io account
As part of the Early Access Program you are required to have an Quay.io Account in order to be able to
pull the Red Hat Developer Hub container image.

 1. Make sure you have been granted acess to the RHDH Organization. You can check that by trying https://quay.io/organization/rhdh
 > If you are not able to acess this Quay.io org, please ask for it by sending an email to `rhdh-interest@redhat.com` and fill out this Google Form https://forms.gle/hTnjWuV84DJbRT5Q7 

 2. Next, download a pull secret from you quay.io account and apply to the Openshift Project where Dev Hub will be installed to.
  * From quay.io access `Account Settings -> Generate Encrypted Password -> Kubernetes Secret -> Donwload <username>-secret.yml
  * change the name of the Secret to `rhdh-pull-secret`
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: rhdh-pull-secret #<--- has to be named exactly this way
  data:
    .dockerconfigjson: your quay secret here
  type: kubernetes.io/dockerconfigjson
  ```

  * finaly apply it
  ```
  oc apply -f username-secret.yml -n rhdh
  ```

### Install using the Red Hat Developer Hub Helm Chat from the Openshift Helm Chart marketplace
## Initial configuration
 * Create a new Project Namespaces named `rhdh`. We'll be deploying Dev Hub in this namespace.

 * Use the Developer Hub Helm Chart to create an instance and wait for the Postgres and DevHub PODs to come up to a healthy state.

  > :exclamation: During the intial setup its recommended to set the Dev Hub logging to DEBUG mode. To do that simply run the following command:

  ``` sh
  oc set env deployment developer-hub LOG_LEVEL=debug -n rhdh
  ```

### Main Config Secret

Create a secret containing all the Github Org/App info as key/value map

``` sh
#source all the env vars collected above
source ./env.sh
#delete the existing one
oc delete secret rhdh-secret --ignore-not-found=true -n rhdh
#create a new one
oc create secret generic rhdh-secret -n rhdh \
--from-literal=GITHUB_ORG_URL=$GITHUB_ORG_URL \
--from-literal=GITHUB_APP_APP_ID=$GITHUB_APP_APP_ID \
--from-literal=GITHUB_APP_CLIENT_ID=$GITHUB_APP_CLIENT_ID \
--from-literal=GITHUB_APP_CLIENT_SECRET=$GITHUB_APP_CLIENT_SECRET \
--from-literal=GITHUB_APP_PRIVATE_KEY=$GITHUB_APP_PRIVATE_KEY \
--from-literal=GITHUB_APP_WEBHOOK_URL=$GITHUB_APP_WEBHOOK_URL \
--from-literal=GITHUB_APP_WEBHOOK_SECRET=$GITHUB_APP_WEBHOOK_SECRET \
--from-literal=GITHUB_ENABLED=true
```

Your secret should look like this.

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: rhdh-secret
  namespace: rhdh
data:
  GITHUB_APP_APP_ID: 
  GITHUB_APP_CLIENT_ID: 
  GITHUB_APP_CLIENT_SECRET: 
  GITHUB_APP_PRIVATE_KEY: 
  GITHUB_APP_WEBHOOK_URL: 
  GITHUB_APP_WEBHOOK_SECRET: 
  GITHUB_ENABLED: 
  GITHUB_ORG_URL: 
type: Opaque
```
### Additional RHDH Config Map
This ConfigMap inject adtional config to the DevHub instance.

 * create Config Map a named `app-config-rhdh` with the following content.
  
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: app-config-rhdh
  namespace: rhdh
immutable: false
data:
  app-config-rhdh.yaml: |
    app:
      title : Red Hat Developer Hub    
    integrations:
      github:
        - host: github.com # if using Hosted Enterprise GitHub, replace this by your domain URL
          #----- 
          # !!! if integrating with a Hosted GitHub Enterprise instance add the following next line !!!
          #apiBaseUrl: https://your-github-enterprie-domain/api/v3
          #----- 
          apps:
            - appId: ${GITHUB_APP_APP_ID} 
              clientId: ${GITHUB_APP_CLIENT_ID} 
              clientSecret: ${GITHUB_APP_CLIENT_SECRET}
              webhookUrl: ${GITHUB_APP_WEBHOOK_URL}
              webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
              privateKey: |
                ${GITHUB_APP_PRIVATE_KEY}
                
    auth:
      environment: development
      providers:
        github:
          development:
            #----- 
            # !!! if integrating with a Hosted GitHub Enterprise instance add the following next line !!!
            #enterpriseInstanceUrl: https://your-github-enterprie-domain
            #----- 
            clientId: ${GITHUB_APP_CLIENT_ID}
            clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          

    catalog:
      providers:
        githubOrg:
          default:
            id: development
            orgUrl: ${GITHUB_ORG_URL}        

    enabled:
      github: ${GITHUB_ENABLED} 
      githubOrg: true   
```

### Upgrade the Dev Hub Helm Char Values
Upgrade the Dev Hub instance passing additional values in order to load our additional ConfigMap.

```yaml
global:
  clusterRouterBase: apps.YOUR_CLUSTER_DOMAIN # UPDATE THSI WITH YOUR CUSTER DOMAIN!
route:
  enabled: true
  host: '{{ .Values.global.host }}'
  path: /
  tls:
    enabled: true
    insecureEdgeTerminationPolicy: Redirect
    termination: edge
  wildcardPolicy: None
upstream:
  backstage:
    appConfig:
      app:
        baseUrl: 'https://{{- include "janus-idp.hostname" . }}'
      backend:
        baseUrl: 'https://{{- include "janus-idp.hostname" . }}'
        cors:
          origin: 'https://{{- include "janus-idp.hostname" . }}'
        database:
          connection:
            password: '${POSTGRESQL_ADMIN_PASSWORD}'
            user: postgres
    args:
      - '--config '
      - app-config.yaml
      - '--config'
      - app-config.example.yaml
      - '--config'
      - app-config.example.production.yaml
    command:
      - node
      - packages/backend
    containerPorts:
      backend: 7007
    extraAppConfig:
      - configMapRef: app-config-rhdh # DOUBLE CHECK THIS CM NAME!!!
        filename: app-config-rhdh.yaml
    extraEnvVars:
      - name: POSTGRESQL_ADMIN_PASSWORD
        valueFrom:
          secretKeyRef:
            key: postgres-password
            name: '{{ .Release.Name }}-postgresql'
      - name: NODE_TLS_REJECT_UNAUTHORIZED
        value: '0'     
    extraEnvVarsSecrets:
      - rhdh-secret # DOUBLE CHECK THIS SECRET NAME!!!
    image:
      debug: true
      pullPolicy: Always
      pullSecrets:
        - rhdh-pull-secret
      registry: quay.io
      repository: rhdh/rhdh-hub-rhel9
      tag: 1.0-88
    installDir: /app
    replicas: 1
    revisionHistoryLimit: 10
  clusterDomain: cluster.local
  diagnosticMode:
    args:
      - infinity
    command:
      - sleep
    enabled: false
  ingress:
    enabled: false
    host: '{{ .Values.global.host }}'
    tls:
      enabled: false
  metrics:
    serviceMonitor:
      enabled: false
      path: /metrics
  nameOverride: developer-hub
  networkPolicy:
    enabled: false
  postgresql:
    auth:
      secretKeys:
        adminPasswordKey: postgres-password
        userPasswordKey: password
    enabled: true
    image:
      registry: registry.redhat.io
      repository: rhel9/postgresql-15
      tag: latest
    postgresqlDataDir: /var/lib/pgsql/data/userdata
    primary:
      containerSecurityContext:
        enabled: false
      extraEnvVars:
        - name: POSTGRESQL_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              key: postgres-password
              name: '{{ .Release.Name }}-postgresql'
      persistence:
        enabled: true
        mountPath: /var/lib/pgsql/data
        size: 1Gi
      podSecurityContext:
        enabled: false
      securityContext:
        enabled: false
  service:
    externalTrafficPolicy: Cluster
    ports:
      backend: 7007
      name: http-backend
      targetPort: backend
    sessionAffinity: None
    type: ClusterIP
  serviceAccount:
    automountServiceAccountToken: true
    create: false
```

Now wait for the RHDH POD to be restarted and test it.

### Import Sample GPTs
 1. Fork this repo into your GitHub org: https://github.com/redhat-na-ssa/software-templates
 2. Add a new location entry in the `apr-config-rhdh` Config Map.

```yaml
      locations:
        - type: url
          target: https://github.com/YOUR_GITHUB_ORG/software-templates/blob/main/showcase-templates.yaml
```

 3. Save the CM
 4. Restart the DevHub POD
 5. In the DevHub dashboad, got to Create (left menu) and see all the imported **sample** Golden Path Templates

### Create a sample Catalog Item based on the SpringBoot backend Template
 1. In the Golden Path Templates page, seach for `SpringBoot`
 2. Click on `Choose` and start filling out the Wizard prompts
 3. At the end hit the `Create` button
 4. Create -> select the App entity -> About card -> Hit refresh icon, Refresh the page
 5. Check all the new tab that shows up in the screen
 6. Go back to the Catalog

### Enabling Kubernetes Plugin

> We assume all the components there is being integrated with Developer Hub is installed in the same cluster hosting RHDH instance. If some of the compoenents (eg. ArgoCD, ACM, ACS, etc) is running on a different cluster,
> you may need to repeat this step on other clusters.

 1. create a new SA
 > using the CLI

``` sh
oc create sa rhdh-k8s-plugin -n rhdh
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rhdh-k8s-plugin
  namespace: rhdh
```

 2. Create a Secret Token and bind it to the ServiceAccount
 > using the CLI

``` sh
oc create token rhdh-k8s-plugin -n rhdh
```
> Copy the token from the output and save it for later!

> using the Openshift Administrator Console `Import YAML` (click the `+` icon located at the far top-right menu)
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: rhdh-k8s-plugin-secret
  namespace: rhdh
  annotations:
    kubernetes.io/service-account.name: rhdh-k8s-plugin
```

 3. Create a ClusterRole
 > use the Openshift Administrator Console `Import YAML` (click the `+` icon located at the far top-right menu)

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1 
metadata:
  name: rhdh-k8s-plugin
rules:
  - verbs: 
      - get
      - watch
      - list 
    apiGroups:
      - '' 
    resources:
      - pods
      - pods/log
      - services
      - configmaps 
      - limitranges
  - verbs: 
      - get
      - watch
      - list 
    apiGroups:
      - metrics.k8s.io 
    resources:
      - pods 
  - verbs:
      - get
      - watch 
      - list
    apiGroups: 
      - apps
    resources:
      - daemonsets
      - deployments 
      - replicasets 
      - statefulsets
  - verbs: 
      - get
      - watch
      - list 
    apiGroups:
      - autoscaling 
    resources:
      - horizontalpodautoscalers 
  - verbs:
      - get
      - watch 
      - list
    apiGroups:
      - networking.k8s.io
    resources: 
      - ingresses
  - verbs: 
      - get
      - watch 
      - list
    apiGroups: 
      - batch
    resources: 
      - jobs
      - cronjobs
  - verbs: 
      - get
      - list 
    apiGroups:
      - tekton.dev 
    resources:
      - pipelines
      - pipelineruns 
      - taskruns
  - verbs:
      - get
      - list
    apiGroups:
      - route.openshift.io
    resources:
      - routes
  - verbs:
      - get
      - list
    apiGroups:
      - org.eclipse.che
    resources:
      - checlusters
```

 4. Create a ClusterRoleBinding for the ServiceAccount
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rhdh-k8s-plugin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rhdh-k8s-plugin
subjects:
  - kind: ServiceAccount
    name: rhdh-k8s-plugin
    namespace: rhdh
```

 5. Create a new Secret with the Cluster info
 > using the CLI

``` sh
oc create secret generic backstage-k8s-plugin-secret -n rhdh \
--from-literal=K8S_CLUSTER_NAME='development-cluster' \
--from-literal=K8S_CLUSTER_TOKEN='<ServiceAccount token from step 2>' \
--from-literal=K8S_CLUSTER_URL=$K8S_CLUSTER_API

```

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: backstage-k8s-plugin-secret
  namespace: rhdh
stringData:
  K8S_CLUSTER_NAME: 'development-cluster'
  K8S_CLUSTER_TOKEN: 'comes from the secret token create at step #2 (k8s-plugin-secret)'
  K8S_CLUSTER_URL: 'copy your cluster api url (with the :6343)'
type: Opaque
``` 

 6. Enable the Kubernetes Plugin in the DevHub Config Map
   * Update the app-config-rhdh Config Map by ading this section (at the same level and `auth:`)
  
```yaml
data:
  app-config-rhdh.yaml: |

    kubernetes:
      clusterLocatorMethods:
        - clusters:
          - authProvider: serviceAccount
            name: ${K8S_CLUSTER_NAME}
            serviceAccountToken: ${K8S_CLUSTER_TOKEN}
            url: ${K8S_CLUSTER_URL}
            skipTLSVerify: true
          type: config
      customResources:
        # to view the Tekton PipelineRuns list in the side panel 
        # and to view the latest PipelineRun status in the Topology node decorator:
        - group: 'tekton.dev'  #<---
          apiVersion: 'v1beta1'
          plural: 'pipelines'
        - group: 'tekton.dev'
          apiVersion: 'v1beta1'
          plural: 'pipelineruns'
        - group: 'tekton.dev'
          apiVersion: 'v1beta1'
          plural: 'taskruns'
        # to view the edit code decorator:
        - group: 'org.eclipse.che'
          apiVersion: 'v2'
          plural: 'checlusters'
        # to view the OpenShift route
        - group: 'route.openshift.io'
          apiVersion: 'v1'
          plural: 'routes'
      serviceLocatorMethod:
          type: multiTenant

    enabled:
      github: ${GITHUB_ENABLED} 
      githubOrg: true
      kubernetes: true #<--- Here!
```

 7. Upgrade the DevHub Helm values by adding the secret name `backstage-k8s-plugin-secret` under `extraEnvVarsSecrets:`

```yaml
    extraEnvVarsSecrets:
      - rhdh-secret
      - backstage-k8s-plugin-secret # from step #5
```

### Enabling Openshift GitOps (ArgoCD) Plugin
 * Plugin Docs: https://access.redhat.com/documentation/en-us/red_hat_plug-ins_for_backstage/2.0

### Enabling OCM Plugin

 * Plugin (upstream) doc: https://github.com/janus-idp/backstage-plugins/blob/main/plugins/ocm/README.md
 * 

### Enabling Quay Container Registry Plugin

 * Plugin (upstream) doc: https://github.com/janus-idp/backstage-plugins/blob/main/plugins/quay/README.md
 * 

---

## Onboarding a Sample Application Entity

---

## Creating a sample Golden Path Template

> Developer Hub (based on janus-idp) Plugins and Custom Actions repository: https://github.com/janus-idp/backstage-plugins

 * Create a new container image repo on Quay Container Registry: https://github.com/janus-idp/backstage-plugins/blob/main/plugins/quay-actions/examples/templates/01-quay-template.yaml

---

# Openshift DevSpaces 
> DevSpaces documentation: https://access.redhat.com/documentation/en-us/red_hat_openshift_dev_spaces/3.8

 1. from the Openshift Operator Hub install
    * Openshift DevSpaces
    * Kubernetes Image Puller 
 2. Create new Namepsace (not project) named `openshift-devspaces`
``` yaml
kind: Namespace
apiVersion: v1
metadata:
  name: openshift-devspaces
  labels:
    kubernetes.io/metadata.name: openshift-devspaces
spec:
  finalizers:
    - kubernetes
```

 3. Create a new **Red Hat OpenShift Dev Spaces instance Specification** resource definition (`CheCluster`)
``` yaml
apiVersion: org.eclipse.che/v2
kind: CheCluster
metadata:
  name: devspaces
  namespace: openshift-devspaces
spec:
  components:
    cheServer:
      debug: false
      logLevel: INFO
    dashboard: {}
    devWorkspace:
      runningLimit: '5'
    devfileRegistry: {}
    imagePuller:
      enable: true
      spec: {}
    metrics:
      enable: true
    pluginRegistry:
      openVSXURL: 'https://open-vsx.org'
  containerRegistry: {}
  devEnvironments:
    startTimeoutSeconds: 600
    secondsOfRunBeforeIdling: -1
    maxNumberOfWorkspacesPerUser: 5
    containerBuildConfiguration:
      openShiftSecurityContextConstraint: container-build
    defaultEditor: che-incubator/che-code/insiders
    maxNumberOfRunningWorkspacesPerUser: 5
    defaultNamespace:
      autoProvision: true
      template: <username>-devspaces
    secondsOfInactivityBeforeIdling: -1
    storage:
      # perWorkspaceStrategyPvcConfig:
      #   claimSize: 10Gi
      #   storageClass: gp3-csi
      pvcStrategy: per-workspace
  gitServices: {}
  networking:
    auth:
      gateway:
        configLabels:
          app: che
          component: che-gateway-config

```

### Integrate Openshift DevSpaces and Github (for access tokens)

Create a GitHub **OAuth** application to enable Dev Spaces to seamlessly push code changes to the repository for new components created in Red Hat Developer Hub.  

``` sh
open "https://$GITHUB_HOST_DOMAIN/settings/applications/new?oauth_application[name]=$GITHUB_ORGANIZATION-devspaces&oauth_application[url]=https://devspaces.apps$OPENSHIFT_CLUSTER_INFO&oauth_application[callback_url]=https://devspaces.apps$OPENSHIFT_CLUSTER_INFO/api/oauth/callback"
```

Copy the App Id and the Secret Id and set the `GITHUB_DEV_SPACES_CLIENT_ID` and `GITHUB_DEV_SPACES_CLIENT_SECRET` environment variables with its values. Then copy&paste the script content into a file, make it executable (`chmod +x `) and execute it (make sure you are logged in the riwght cluster!).

``` sh
!#/bin/sh
export GITHUB_DEV_SPACES_CLIENT_ID=
export GITHUB_DEV_SPACES_CLIENT_SECRET=
export DEV_SPACES_NAMESPACE='openshift-devspaces'

oc delete secret github-oauth-config --ignore-not-found=true -n openshift-devspaces
oc create secret generic github-oauth-config -n openshift-devspaces \
--from-literal=id=$GITHUB_DEV_SPACES_CLIENT_ID \
--from-literal=secret=$GITHUB_DEV_SPACES_CLIENT_SECRET

oc label secret github-oauth-config -n $DEV_SPACES_NAMESPACE \
--overwrite=true app.kubernetes.io/part-of=che.eclipse.org app.kubernetes.io/component=oauth-scm-configuration

oc annotate secret github-oauth-config -n $DEV_SPACES_NAMESPACE \
--overwrite=true che.eclipse.org/oauth-scm-server=github 

# if using Github Enterprise, also add these two additional annotations
#oc annotate secret github-oauth-config -n $DEV_SPACES_NAMESPACE \
#--overwrite=true che.eclipse.org/scm-server-endpoint=github_enterprise_server_url che.eclipse.org/scm-github-disable-subdomain-isolation="false"
```

 * On your Github Project's repo, edit the `catalog-info.yaml` descriptor and add this `slug` (Backstage terminology)
```yaml
#...
metadata:
#...
  annotations:
    github.com/project-slug: <<organization>>/<<repo>>
#...
links:
  - url: https://devspaces.apps.cluster-domain/#https://github.com/organization/app-repo-name
    title: OpenShift Dev Spaces (VS Code)
    icon: web

```

 

---

# Install Red Hat Quay Container Registry (if not yet installed)
> ❗Prerequisites : S3 bucket or ODF
1. Install the quay operator
2. Create the configmap file called configmap.yaml and copy the contents below

```yaml
REGISTRY_TITLE: Red Hat Product Demo System Quay
SERVER_HOSTNAME: quay-registry.<<Cluster URL>> #e.g quay-registry.apps.cluster-dnqwv.dnqwv.sandbox2019.opentlc.com
EXTERNAL_TLS_TERMINATION: true
SUPER_USERS:
- quayadmin
FEATURE_USER_INITIALIZE: true
BROWSER_API_CALLS_XHR_ONLY: false
DISTRIBUTED_STORAGE_CONFIG:
  s3Storage:
    - S3Storage
    - host: s3.amazonaws.com
      s3_access_key: <<AWS ACCESS KEY>>
      s3_secret_key: <<AWS SECRET KEY>>
      s3_bucket: <<AWS Bucket Name>>
      storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
    - s3Storage
```
3. Create the project namespaces
```sh
  oc project quay
  oc create secret generic config-bundle-secret --from-file config.yaml=./config.yaml
```
4. Create the quay registry CR in the quay namespace

```yaml
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: rhdh-quay-registry
  namespace: quay
spec:
  components:
    - kind: clair
      managed: true
    - kind: postgres
      managed: true
    - kind: objectstorage
      managed: false
    - kind: redis
      managed: true
    - kind: horizontalpodautoscaler
      managed: true
    - kind: route
      managed: true
    - kind: mirror
      managed: false
    - kind: monitoring
      managed: true
    - kind: tls
      managed: true
    - kind: quay
      managed: true
    - kind: clairpostgres
      managed: true
  configBundleSecret: config-bundle-secret
```

5. Once quay is up open the quay regsitry url via route created.
6. Create an account with user/password.
7. Create an org name `rhdh-demo`
8. Create an application and generate bearer token.
9. Testing Quay Registry
  ```sh
     podman pull docker.io/redis
     podman login -u <<user>> <<quay-url>>   # <<user>>/<<password>> is from setup 6 and <<quay-url>> is from the installed quay route
     podman tag docker.io/redis <<quay-url>>/<<orgname>>/redis
     podman push <<quay-url>>/<<orgname>>/redis
     aws configure
     aws s3 ls <<bucket-name>> --recursive
  ```

# Configuring OCM (Open Cluster Management) Plugin
> ❗Prerequisites : ACM must be installed on the cluster

#### Note: Even after installing OCM plugin it will take an hour to sync the OCM cluster details under clusters tab.

If ACM is installed in a different cluster then we need to perform the following on HUB cluster. If installed in the same cluster then move directly to step 7.

1. create a new SA
 > using the CLI

``` sh
oc create sa acm-rhdh-k8s-plugin -n rhdh
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: acm-rhdh-k8s-plugin
  namespace: rhdh
```

 2. Create a Secret Token and bind it to the ServiceAccount
 > using the CLI

``` sh
oc create token acm-rhdh-k8s-plugin -n rhdh
```
> Copy the token from the output and save it for later!

> using the Openshift Administrator Console `Import YAML` (click the `+` icon located at the far top-right menu)
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: acm-rhdh-k8s-plugin-secret
  namespace: rhdh
  annotations:
    kubernetes.io/service-account.name: amc-rhdh-k8s-plugin
```

 3. Create a ClusterRole
 > use the Openshift Administrator Console `Import YAML` (click the `+` icon located at the far top-right menu)

 ```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: acm-rhdh-k8s-plugin
rules:
  - apiGroups:
      - cluster.open-cluster-management.io
    resources:
      - managedclusters
    verbs:
      - get
      - watch
      - list
  - apiGroups:
      - internal.open-cluster-management.io
    resources:
      - managedclusterinfos
    verbs:
      - get
      - watch
      - list
 ```

4. Create a ClusterRoleBinding for the ServiceAccount
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: acm-rhdh-k8s-plugin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: acm-rhdh-k8s-plugin
subjects:
  - kind: ServiceAccount
    name: acm-rhdh-k8s-plugin
    namespace: rhdh
```
5. Create a new Secret with the Cluster info for ACM
 > using the CLI

``` sh
oc create secret generic backstage-k8s-plugin-secret -n rhdh \
--from-literal=ACM_K8S_CLUSTER_NAME='<ACM_HUB_ClusterName>>' \
--from-literal=ACM_K8S_CLUSTER_TOKEN='<ServiceAccount token from step 2>' \
--from-literal=ACM_K8S_CLUSTER_URL='<<HUB API URL for Openshift>>'

```

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: acm-backstage-k8s-plugin-secret
  namespace: rhdh
stringData:
  ACM_K8S_CLUSTER_NAME: 'hub-cluster'
  ACM_K8S_CLUSTER_TOKEN: 'comes from the secret token create at step #2 (k8s-plugin-secret)'
  ACM_K8S_CLUSTER_URL: 'copy your cluster api url (with the :6343)'
type: Opaque
```

6. Update app-config-rhdh.yaml to include the plugin 
   > Note : The following changes needs to be installed on cluster where RHDH is installed.
   > Add the following under Catalog.providers section

   ```yaml
   ocm:
     env: 
        name: ${ACM_K8S_CLUSTER_NAME} 
        url: ${ACM_K8S_CLUSTER_URL}
        serviceAccountToken: ${ACM_K8S_CLUSTER_TOKEN}
        schedule: # optional;proper same options as in TaskScheduleDefinition
              # supports cron, ISO duration, "human duration" as used in code
              frequency: { seconds: 10 }
              # supports ISO duration, "human duration" as used in code
              timeout: { seconds: 60 }  
   ```

  > Under enabled section of yaml please add the following.

  ```yaml
       enabled:
          ocm: true
  ```
  > Upgrade the DevHub Helm values by adding the secret name `acm-backstage-k8s-plugin-secret` under `extraEnvVarsSecrets:`

  ```yaml
      extraEnvVarsSecrets:
        - rhdh-secret
        - backstage-k8s-plugin-secret
        - acm-backstage-k8s-plugin-secret # from step #5
  ```

7. Update app-config-rhdh.yaml only when ACM is installed on same cluster as RHDH If not please ignore this step.
  > Add the following under Catalog.providers section
  ```yaml
      ocm:
          env:
            kubernetesPluginRef: ${K8S_CLUSTER_NAME}
            schedule: # optional;proper same options as in TaskScheduleDefinition
              # supports cron, ISO duration, "human duration" as used in code
              frequency: { seconds: 10 }
              # supports ISO duration, "human duration" as used in code
              timeout: { seconds: 60 }    
  ```
  > Under enabled section of yaml please add the following.

  ```yaml
       enabled:
          ocm: true
  ```


# Configuring Quay plugin.
> ❗Prerequisites : Quay Registry is available and configured with organization and bearer token is available.

1. Create a secret to store quay url and quay bearer token

```sh
   export QUAY_URL=<<Quay URL>>
   export QUAY_BEARER_TOKEN=<<Quay bearer token>>

   oc create secret generic quay-secret -n rhdh --from-literal QUAY_URL=${QUAY_URL} --from-literal QUAY_BEARER_TOKEN=${QUAY_BEARER_TOKEN}
```

2. Add the following to app-config-rhdh.yaml for configuring quay plugin
   > Note : The quay and proxy should be aligned to auth or any plugin root level

```yaml

  quay:
      uiUrl: ${QUAY_URL}

  proxy:
    endpoints:  
     '/quay/api':
          target: ${QUAY_URL}
          headers:
            X-Requested-With: 'XMLHttpRequest'
            Authorization: "Bearer ${QUAY_BEARER_TOKEN}"
          changeOrigin: true
          # Change to "false" in case of using self hosted quay instance with a self-signed certificate
          secure: false
```

  Also enable the plugin via the following
```yaml
    enabled:
      quay: true

```

3. Upgrade the helm deployment by including this quay-secret under  extraEnvVarsSecrets

```yaml
     extraEnvVarsSecrets:
        - rhdh-secret
        - backstage-k8s-plugin-secret
        - acm-backstage-k8s-plugin-secret
        - quay-secret
```

# Customizing Logo and Themes.
> Note : Logo's can be added with from svg or image. We need a base64 version of the svg/image

1. Get the logo as svg/image and save it as a file e.g logo.txt
2. convert to base64
  ```sh
      cat logo.txt | base64 > logo_base64.txt
  ```
3. prefix the following content `data:image/png;base64,` or  `data:image/svg+xml;base64,` depending on image or svg to base64 file
4. create a secret named logo-secret (key/value secret)
   > Note: Create through console don't use the below yaml this is just for reference

   ```yaml
      kind: Secret
      apiVersion: v1
      metadata:
        name: logo-secret
        namespace: rhdh
      stringData:
        BASE64_EMBEDDED_FULL_LOGO: <<content of logo_base64.txt>>
      type: Opaque
   ``` 
5. Add the following content on app-config-rhdh.yaml

   ```yaml
      app:
        title : Red Hat Developer Hub
        branding:
          fullLogo: ${BASE64_EMBEDDED_FULL_LOGO}
          iconLogo: ${BASE64_EMBEDDED_FULL_LOGO}
        theme:
          light:
            primaryColor: '#38BE8B'
            headerColor1: 'hsl(204 100% 71%)'
            headerColor2: 'color(a98-rgb 1 0 0)'
            navigationIndicatorColor: '#be0000'
          dark:
            primaryColor: '#ab75cf'
            headerColor1: '#0000d0'
            headerColor2: 'rgb(255 246 140)'
            navigationIndicatorColor: '#f4eea9' 
   ```

6. Upgrade the helm deployment by including this logo-secret under  extraEnvVarsSecrets

    ```yaml
        extraEnvVarsSecrets:
            - rhdh-secret
            - backstage-k8s-plugin-secret
            - acm-backstage-k8s-plugin-secret
            - quay-secret
            - logo-secret
    ```    

# Customizing Homepage and TechRadar.    

1. Fork the git project in customers SCM. (https://github.com/redhat-na-ssa/backstage-customization-provider)
2. If remote fork is not allowed in the customer env use the following workaround
     1. Create a new repo under the Org
     2. git clone https://github.com/redhat-na-ssa/backstage-customization-provider
     3. execute the following
       ```sh
          cd backstage-customization-provider
          git remote add enterprise <<Newly created git url>>
          git push -u enterprise main
       ```
3. Update the data.json file under <<Project Root>>/data/home/ for home page url
4. Update the data.json file under <<Project Root>>/data/tech-radar/ for home page url 
5. Perform a s2i deployment to rhdh namespace
6. Edit the existing logo-secret for homepage/tech-radar customization
   > Note : Add the following keys to `logo-secret`
     ```yaml
           HOMEPAGE_DATA_URL: "http://<<servicename>>:8080"
           TECHRADAR_DATA_URL: "http://<<servicename>>:8080/tech-radar"
     ```
7. Add the following proxy config in app-config-rhdh.yaml
   ```yaml
       proxy:
        endpoints:  
          # Other Proxies
          # customize developer hub instance
          '/developer-hub':
            target: ${HOMEPAGE_DATA_URL}
            changeOrigin: true
            # Change to "false" in case of using self hosted cluster with a self-signed certificate
            secure: false
          '/developer-hub/tech-radar':
            target: ${TECHRADAR_DATA_URL}
            changeOrigin: true
            # Change to "false" in case of using self hosted cluster with a self-signed certificate
            secure: false 
   ```
8. Delete the RHDH pod to reload with this new configmap updates.
