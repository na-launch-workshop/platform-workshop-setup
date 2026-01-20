# Developer Experience Workshop Setup

This repository contains the setup instructions to build the Red Hat Launch Workshop.  The workshop module content
is prefixed with `workshop-` and available in [this repository](https://github.com/orgs/na-launch-workshop/repositories).

## Introduction

This workshop is deployed via the [cluster-bootstrapper](https://github.com/na-launch-workshop/platform-cluster-bootstrapper) which
supports both building new clusters and bringing your own cluster.

### Build Your Cluster

TODO: Writeup of instructions on using the cluster-bootstrapper to build a cluster.

### Bring Your Own Cluster

For ease of deployment with this workshop, you can bring your own OpenShift 4 cluster by ordering "AWS with ROSA Open Environment"
from [demo.redhat.com](https://demo.redhat.com).

### Architecture

The workshop is deployed in two phases:

#### 1. [Workshop Infrastructure](#deploy-workshop-infrastructure)

Terraform and Ansible automation deploy Red Hat OpenShift GitOps (ArgoCD) which in turn deploys an app of apps pattern:

- At the root is [infrastructure-orchestrator](https://github.com/na-launch-workshop/platform-charts/tree/main/charts/orchestrator)
- ... which deploys [infrastructure-rollout-controller](https://github.com/na-launch-workshop/platform-charts/tree/main/charts/rollout-controller)
- ... which deploys the [workshop infrastructure Helm charts](https://github.com/na-launch-workshop/platform-charts/tree/main/charts)

The workshop infrastructure Helm charts deploy the following operators:

- Red Hat Single Sign-On (Keycloak)
- Vault
- External Secrets
- GitLab (self-hosted)
- Red Hat Developer Hub

#### 2. [Workshop Template](#deploy-workshop-template)

Launching a one-time setup Developer Hub template for the workshop will deploy another app of app pattern:

- GitLab cloned copy of [workshop-orchestrator](https://github.com/na-launch-workshop/platform-software-templates/blob/main/workshop/developer-experience/skeleton/overlays/dev/patch-values.yaml)
- ... which deploys workshop-rollout-controller
- ... which deploys the operators required for the workshop module content:
  - Red Hat OpenShift Dev Spaces
  - Streams for Apache Kafka
  - Red Hat OpenShift Serverless
  - Red Hat build of OpenTelemetry
  - Minio (for S3 storage for OpenTelemetry)

## Deployment

### Deploy Workshop Infrastructure

1. Clone this Git repository: https://github.com/na-launch-workshop/platform-cluster-bootstrapper

2. Replace [config.yaml](https://github.com/na-launch-workshop/platform-cluster-bootstrapper/blob/main/config.yaml)
in the cloned repository with this example [config.yaml](examples/config.yaml).

For ROSA and HCP installations, the name is the domain prefix of your cluster in [Hybrid Cloud Console](https://console.redhat.com/openshift/cluster-list)
  ```yaml
    - name: rosa-abcde
  ```

The domain is name of your cluster domain (i.e. your OpenShift API endpoint is https://api.rosa-abcde.subdomain.openshiftapps.com)
  ```yaml
      domain: rosa-abcde.subdomain.openshiftapps.com
  ```

You will need to obtain a ROSA token.  If you are using demo.redhat.com's "AWS with ROSA Open Environment", you can log into the provided bastion which
is set up for rosa cli (this is from demo.redhat.com, not your own ROSA token):

  ```sh
  rosa token --generate
  ```
Otherwise, you can obtain a token from the [Hybrid Cloud Console](https://console.redhat.com/openshift/token/rosa).  Red Hat recommends using Red Hat SSO
to authenticate against OCM, but we need to use API tokens for specific components.  See the "Still need access to API tokens to authenticate?" note to obtain a token.

Update the AWS and ROSA credentials:
  ```yaml
      providerCredentials:
        aws_access_key_id: ""
        aws_secret_access_key: ""
        region: "us-east-2"
        rosa_token: ""
  ```

Update the version of the chart as needed.  The latest published version can be found in [Chart.yaml](https://github.com/na-launch-workshop/platform-charts/blob/main/charts/orchestrator/Chart.yaml)
  ```yaml
      workshop:
        chart:
          enabled: true
          name: orchestrator
          version: 0.0.90
  ```

  To install the workshop infrastructure components and configure Developer Hub plugins to access those components (i.e. GitLab,
  OpenShift GitOps, ...), run:

  ```sh
  make configure
  ```

You can check the progress of ArgoCD reconciliation by logging into OpenShift GitOps with the `admin` user and its password from this command:
  ```sh
  oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}'|base64 --decode
  ```

The setup files can be [found here](https://github.com/na-launch-workshop/platform-charts/tree/main/charts/developer_hub/templates) and
toggling on/off individual operators can be [found here](https://github.com/na-launch-workshop/platform-ansible-collections/blob/c5dddfd53af223604856f79811a0d504dc8e404f/bootstrap/workshop/roles/gitops-instance/tasks/main.yaml#L493).

### Deploy Workshop Template

Now that the infrastructure components have been installed, log into Developer Hub as an administrator.
Import the workshop template in Developer Hub by navigating to Catalog -> Self-service -> Register
Existing Component and using [this template](https://github.com/na-launch-workshop/platform-software-templates/blob/main/workshop/developer-experience/template.yaml)

Using the newly imported template will deploy the workshop by writing files to the internal GitLab
where OpenShift GitOps will sync to the repository and build out the operators required.  The
template will also ask the user whether extra capacity for Dev Spaces and Kafka need to be provisioned.
Kicking off off the template writes towards the self-hosted GitLab, and OpenShift GitOps will sync
changes and build out the operators required for end users to use the workshop module content.

## Administration

### Scaling a demo.redhat.com Cluster

If you are using demo.redhat.com to provision a ROSA cluster, it needs to be scaled up to accommodate the myriad of operators installed
in this workshop.  Thus, you can follow these instructions to provision additional nodes.

Log into the provided bastion which is set up for rosa cli.  Then scale up machinepool as follows:
  ```sh 
  $ rosa list cluster
  ID                                NAME        STATE  TOPOLOGY
  2ntuf3b1gnkousjpu38loltvsc8f42u8  rosa-4rctx  ready  Hosted CP
  
  $ rosa list machinepool --cluster=2ntuf3b1gnkousjpu38loltvsc8f42u8
  ID       AUTOSCALING  REPLICAS  INSTANCE TYPE  LABELS    TAINTS    AVAILABILITY ZONE  SUBNET                    DISK SIZE  VERSION  AUTOREPAIR
  workers  No           3/3       m6a.xlarge                         us-east-2a         subnet-060b4677ab2a34bdf  300 GiB    4.17.45  Yes

  $ rosa edit machinepool --replicas=3 workers --cluster=2ntuf3b1gnkousjpu38loltvsc8f42u8
  I: Updated machine pool 'workers' on hosted cluster '2ntuf3b1gnkousjpu38loltvsc8f42u8'
  ```

### Adding Cluster Admins

If you are using ROSA hosted control planes, the OpenShift Cluster Manager manages users of the cluster-admins and dedicated-admins group.
To add a cluster-admin, log onto the provided ROSA bastion and run the following commands, substituting the cluster ID with your own:

  ```sh
  rosa list cluster
  ID                                NAME        STATE  TOPOLOGY
  2lm4hsaqidq67ne30i1q279jlml981vi  rosa-77mtr  ready  Hosted CP
  
  rosa grant user cluster-admin --user=admin --cluster=2lm4hsaqidq67ne30i1q279jlml981vi
  ```

### Accessing Keycloak

Administrators may need to access Keycloak to further configure the workshop.  The route is accessible here:
  ```sh
  oc get route -n keycloak
  ```

The configured username and password can be accessed as follows:
  ```sh
  oc get secret credential-keycloak -n keycloak -o jsonpath='{.data.ADMIN_USERNAME}'|base64 --decode
  oc get secret credential-keycloak -n keycloak -o jsonpath='{.data.ADMIN_PASSWORD}'|base64 --decode
  ```

## Workshop Modules

The [workshop module library](https://github.com/orgs/na-launch-workshop/repositories) are prebuilt module content for the developers,
prefixed with `workshop-`.  The workshop administrators should clone from the public GitHub organization to the self-hosted GitLab
instance in the workshop cluster.

### Adding Module Content

The workshop modules can be hosted in Developer Hub's TechDocs for end users to view.  These modules are imported into GitLab as
follows:

1. The GitLab repository that was deployed contains a root user that can import Git repositories from external GitHub.  Log in with the
root GitLab user (the password can be found in `oc get secret gitlab-gitlab-initial-root-password -n gitlab-system -o yaml`)
2. [Configure GitHub](https://docs.gitlab.com/administration/settings/import_and_export_settings/#configure-allowed-import-sources) as
an allowed import source to GitLab self-managed.
3. In GitLab, navigate to the developers group which is an appropriate RBAC role for end users to view but not merge module content to
the main branch.  Create a new project -> Import project -> Import from GitHub.  Import any of the modules from the content library above.
4. Developer Hub automatically detects valid catalog-info.yaml files from the accompanying repositories and imports them and their
TechDocs content.

## Updates

### Updating Helm Charts

Refer to the [platform-charts](https://github.com/na-launch-workshop/platform-charts/actions) repository for instructions on updating Helm charts.

#### Updating Dev Spaces RBAC

Users in Dev Spaces are granted the `developers` role via the configured [CheCluster CR](https://github.com/na-launch-workshop/platform-charts/blob/main/charts/dev-spaces/templates/che-cluster.yaml)
from the dev-spaces Helm chart.  Depending on workshop content and other operators that are installed, the `developers` role RBAC may
need to be configured, which can be done from the [system_roles Helm chart](https://github.com/na-launch-workshop/platform-charts/tree/main/charts/system_roles/templates).
The workflow to update this Helm chart is the same as above.

### Updating Workshop Content in Developer Hub

The workshop content is built in mkdocs format.  To update the website, fork this Git repository,
[update the docs](https://github.com/na-launch-workshop/platform-software-templates/tree/main/workshop/developer-experience/skeleton/repository/docs)
and upload the new software template.
