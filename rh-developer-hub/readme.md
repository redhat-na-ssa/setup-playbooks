# Red Hat Developer Hub (Early Access Program) Setup instructions

> :exclamation: NOTE: *the following instructions outlined here are not intended for building a production-ready environment! It is intended to build a PoC leveraging Red Hat Openshift DevSecOps offerings.* 

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

# Source Control System and Identity Provider setup
> ❗This is the most important integration required by Developer Hub!

## Openshift Cluster Info

Log in to your OpenShift cluster via the `oc` client.  Set the `OPENSHIFT_CLUSTER_INFO` variable for use later.

``` sh
export OPENSHIFT_CLUSTER_INFO=$(oc cluster-info | head -n 1 | sed 's/^.*https...api//' | sed 's/.6443.*$//' )
```

>❗ tip "Using Linux?"
  If you are using a `Linux` environment, set the alias for the following commands to work:

  ```sh
  alias open="xdg-open"
  ```

For the remaining environment variables, it may be preferable create a `./env.sh` containeing the listed env vars as you complete the setup configuration, then run `source ./env.sh`.

## Create GitHub Organization (if not exist)

Create a new [Github Organization](https://github.com/account/organizations/new?plan=free). This organization will contain the code repositories for the `components` created by Red Hat Developer Hub.

The `GITHUB_ORGANIZATION` environment variable will be set to the name of the organization.

> ❗tip
  You may also use any organization you are a member of, as long as you have the ability to create new repositories within it.

``` sh
export GITHUB_ORGANIZATION=
export GITHUB_ORG_URL=https://github.com/$GITHUB_ORGANIZATION
```

## Set Up GitHub Application

1. Create a new GitHub Application to use the `Git WebHooks` functionality in this demo.  The required field will be populated, and correct permissions set.

    ``` sh
    open "https://github.com/organizations/$GITHUB_ORGANIZATION/settings/apps/new?name=$GITHUB_ORGANIZATION-rhdh-app&url=https://janus-idp.io/blog&webhook_active=true&public=false&administration=write&checks=write&actions=write&contents=write&statuses=write&vulnerability_alerts=write&dependabot_secrets=write&deployments=write&discussions=write&environments=write&issues=write&packages=write&pages=write&pull_requests=write&repository_hooks=write&repository_projects=write&secret_scanning_alerts=write&secrets=write&security_events=write&workflows=write&webhooks=write"
    ```

 * Remember to fill out the following fields:
  * Callback URL: `https://developer-hub-rhdh.apps.your-cluster-domain/api/auth/github/handler/frame`
  * Webhook URL: `https://developer-hub-rhdh.apps.your-cluster-domain/`
  * Webhook Secret: `a radom string` (save it to use later in the DevHub config!)

2. Set the `GITHUB_APP_ID` and `GITHUB_APP_CLIENT_ID` environment variables to the App ID  and App Client ID, respectively. Generate a new client secret and set the `GITHUB_APP_CLIENT_SECRET` environment variable.  Then, generate a `Private Key` for this app and **download** the private key file.
    ``` sh
    export GITHUB_APP_ID=
    export GITHUB_APP_CLIENT_ID=
    export GITHUB_APP_CLIENT_SECRET=
    export GITHUB_APP_PRIVATE_KEY=$(cat /path/to/github-app-key.pem)
    ```

    ![Organization Client Info](assets/org-client-info.png)

3. Go to the `Install App` table on the left side of the page and install the GitHub App that you created for your organization.

    ![Install App](assets/org-install-app.png)

## Create Github **OAuth** Applications :x: `Skip this step for now. We'll do this in later phases.`

In this section we'll create and setup **two different** GitHub **OAuth** Applications that will be later integrated with DevSpaces and RHSSO (Keycloak).

### RHSSO Access Manager
Create an GitHub OAuth application in order to use GitHub as an Identity Provider for Red Hat Developer Hub.

``` sh
open "https://github.com/settings/applications/new?oauth_application[name]=$GITHUB_ORGANIZATION-identity-provider&oauth_application[url]=https://rhdh.apps$OPENSHIFT_CLUSTER_INFO&oauth_application[callback_url]=https://keycloak-rhdh.apps$OPENSHIFT_CLUSTER_INFO/auth/realms/rhdh/broker/github/endpoint"
```

Set the `GITHUB_KEYCLOAK_CLIENT_ID` and `GITHUB_KEYCLOAK_CLIENT_SECRET` environment variables with the values from the OAuth application.

``` sh
export GITHUB_KEYCLOAK_CLIENT_ID=
export GITHUB_KEYCLOAK_CLIENT_SECRET=
```

![Get Client ID](assets/client-info.png)

### DevSpaces
Create a **second** GitHub OAuth application to enable Dev Spaces to seamlessly push code changes to the repository for new components created in Red Hat Developer Hub.  

``` sh
open "https://github.com/settings/applications/new?oauth_application[name]=$GITHUB_ORGANIZATION-dev-spaces&oauth_application[url]=https://devspaces.apps$OPENSHIFT_CLUSTER_INFO&oauth_application[callback_url]=https://devspaces.apps$OPENSHIFT_CLUSTER_INFO/api/oauth/callback"
```

Set the `GITHUB_DEV_SPACES_CLIENT_ID` and `GITHUB_DEV_SPACES_CLIENT_SECRET` environment variables with the values from the OAuth application.

``` sh
export GITHUB_DEV_SPACES_CLIENT_ID=
export GITHUB_DEV_SPACES_CLIENT_SECRET=
```

---

# Red Hat Developer Hub
## Installation
### Quay.io account
As part of the Early Access Program you are required to have an Quay.io Account in order to be able to
pull the Red Hat Developer Hub container image.

 1. Make sure you have been granted acess to the RHDH Organization. You can check that by trying https://quay.io/organization/rhdh
 > If you are not able to acess this Quay.io org, please ask for it by sending an email to `	rhdh-interest@redhat.com` and fill out this Google Form https://forms.gle/hTnjWuV84DJbRT5Q7 

 2. Next, download a pull secret from you quay.io account and apply to the Openshift Project where Dev Hub will be installed to.
  * From quay.io access `Account Settings -> Generate Encrypted Password -> Kubernetes Secret -> Donwload <username>-secret.yml
  * change the name of the Secret to `quay-pull-secret`
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: quay-pull-secret #<--- has to be named exactly this way
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
        - host: github.com
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
            clientId: ${GITHUB_RHDH_APP_CLIENT_ID}
            clientSecret: ${GITHUB_RHDH_APP_CLIENT_SECRET}

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
      - pipelineruns 
      - taskruns
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
--from-literal=K8S_CLUSTER_URL=$(oc cluster-info | head -n 1 |  sed 's/^.*https/https/')

```

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: backstage-k8s-plugin-secret
  namespace: rhdh
data:
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
        - apiVersion: v1beta1
          group: tekton.dev
          plural: pipelineruns
        - apiVersion: v1beta1
          group: tekton.dev
          plural: taskruns
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

## Onboarding a Sample Application Entity

## Creating a sample Golden Path Template

---

# Setup Openshift DevSpaces to use Github as iDP (using OIDC)
All you have to do is create a secret properly annotade and the DevSpaces Operator takes care of the configuration.

``` sh
export DEV_SPACES_NAMESPACE='openshift-devspaces'

oc delete secret github-oauth-config --ignore-not-found=true -n openshift-devspaces
oc create secret generic github-oauth-config -n openshift-devspaces \
--from-literal=id=$GITHUB_DEV_SPACES_CLIENT_ID \
--from-literal=secret=$GITHUB_DEV_SPACES_CLIENT_SECRET

oc label secret github-oauth-config -n $DEV_SPACES_NAMESPACE \
--overwrite=true app.kubernetes.io/part-of=che.eclipse.org app.kubernetes.io/component=oauth-scm-configuration

oc annotate secret github-oauth-config -n $DEV_SPACES_NAMESPACE \
--overwrite=true che.eclipse.org/oauth-scm-server=github
```


---

# Install Red Hat Quay Container Registry (if not yet installed)

