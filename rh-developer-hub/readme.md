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

# Source Control System setup
> ❗This is the most important integration required by Developer Hub!

## GitHub
### Identity Provider (OIDC) setup

## Gitlab
### Identity Provider (OIDC) setup

# Red Hat Developer Hub
## Installation

## Initial configuration

## Onboarding a Sample Application Entity

## Creating a sample Golden Path Template
