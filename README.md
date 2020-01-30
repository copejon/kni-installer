
## KNI installer

This repository contains knictl, the CLI tool for the Akraino KNI deployment. Along with blueprints [https://gerrit.akraino.org/r/#/admin/projects/kni/blueprint-pae](https://gerrit.akraino.org/r/#/admin/projects/kni/blueprint-pae) repository, it will allow to deploy Edge sites in a declarative way.

## Dependencies

You will need to create a Red Hat account on [https://www.redhat.com/wapps/ugc/register.html](https://www.redhat.com/wapps/ugc/register.html), then login with your account into [https://cloud.redhat.com/openshift](https://cloud.redhat.com/openshift). This is needed to have download access to the OpenShift installer artifacts.
After that, you will need to get the Pull Secret from
[https://cloud.redhat.com/openshift/install/metal/user-provisioned](https://cloud.redhat.com/openshift/install/metal/user-provisioned) - Copy Pull Secret link.

## Build System Requirements

Software: gcc,git,go


## How to build

First the `knictl` binary needs to be produced. For that you just execute make with the following syntax:

    make build

This will produce a `knictl` tarball file that can be extracted, and will contain the knictl binary, and another dependencies.

## Prerequisites
Before starting the deployment, some configuration steps are needed on the installer host. A directory on $HOME/.kni needs to be created, and this directory needs to contain:
 - **pull-secret.json**: This needs to contain the pull secret that has been described in the previous step. Please copy the secret literally, without adding extra newlines, whitespaces, etc
 - **id_rsa.pub**: This is a public ssh key, that will be used to access the nodes by SSH. If there is no key present there, a new ssh keypair will be generated by the CLI tool.

In case of AWS deployments, create a $HOME/.aws directory, with a **credentials** file on it. This file needs to have the following content:
[default]
aws_access_key_id=xxxx
aws_secret_access_key=xxxx

Please look at [https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-account.html](https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-account.html)

In the case of Google Cloud Platform deployments, create a $HOME/.gcp directory, with a **service account** file on it. This file needs to be named as **osServiceAccount.json** and have something like: 

{
  "type": "service_account",
  "project_id": "openshift-gce-devel",
  "private_key_id": "xxxxxxxxxxx",
  "private_key": "xxxxxxxxxxxxxx",
  "client_email": "xxxxx@xxxx.xxx",
  "client_id": "xxxxx",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/xxxxxxxx@xxxxxxxxx.xxx"
}

To get this service account JSON file, you will need to create an account in GCP, and get it in the APIs & Services section. Take into account that a service account is always coupled to a specific project. That project name will have to be the same in this file as well as in the install-config that will be configured as part of our site configuration. 

In the case of libvirt, a helper script can be executed to prepare the installer for acting as a virthost. You can execute **utils/prep_host.sh** script to properly configure your server to install OpenShift on libvirt.

In the case of baremetal, please check [https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html) in order to prepare your environment for the deployment.

## Structure of a site
In order to deploy a blueprint, you need to create a repository with a site. The site configuration is based in [kustomize](https://github.com/kubernetes-sigs/kustomize), and needs to use our blueprints as base, referencing that properly. A sample site can be seen on[https://github.com/yrobla/kni-site](https://github.com/yrobla/kni-site) . Site needs to have this structure:

    ├── 00_install-config
    │   ├── install-config.name.patch.yaml
    │   ├── install-config.patch.yaml
    │   ├── kustomization.yaml
    │   └── site-config.yaml
    ├── 01_cluster-mods
    │   ├── kustomization.yaml
    │   ├── manifests
    │   └── openshift
    ├── 02_cluster-addons
    │   └── kustomization.yaml
    └── 03_services
        └── kustomization.yaml

***00_install-config***
This repository will contain the basic settings for the site, including the base blueprint/profile, and the site name/domain. The following files are needed:

**kustomization.yaml**: key file, where it will contain a link to the used blueprint/profile, and a reference to the used patches to customize the site

    bases:
    - git::https://gerrit.akraino.org/r/kni/blueprint-pae.git//profiles/production.aws/00_install-config
    patches:
    - install-config.patch.yaml
    patchesJson6902:
    - target:
          version: v1
          kind: InstallConfig
          name: cluster
       path: install-config.name.patch.yaml
    transformers:
    - site-config.yaml
The entry in bases needs to reference the blueprint being used (in this case blueprint-pae), and the profile install-config file (in this case production.aws/00_install-config). The other entries need to be just written literally.

**install-config.patch.yaml** is a patch to modify the domain from the base blueprint. You need to customize with the domain you want to give to your site.

    apiVersion: v1
    kind: InstallConfig
    metadata:
      name: cluster
    baseDomain: devcluster.openshift.com

**install-config.name.patch.yaml** is a patch to modify the site name from the base blueprint. You need to customize with the name you want to give to your site.

    - op: replace
      path: "/metadata/name"
      value: kni-site
**site-config.yaml** needs to be a dummy file, copied literally:

    apiVersion: kni.akraino.org/v1alpha1
    kind: SiteConfig
    metadata:
      name: notImportantHere
    config: {}

***01_cluster_mods***
This is the directory that will contain all the customizations for the basic cluster deployment. You could create patches for modifying number of masters/workers, network settings... everything that needs to be modified on cluster deployment time. It needs to have a basic **kustomization.yaml** file, that will reference the same level file for the blueprint. And you could create additional patches following kustomize syntax:

    bases:
    - git::https://gerrit.akraino.org/r/kni/blueprint-pae.git//profiles/production.aws/01_cluster-mods

**02_cluster_addons** and **03_services** follow same structure as 01_cluster_mods, but in this case is for adding additional workloads after cluster deployment. They also need to have a **kustomization.yaml** file that references the file of the same level for the blueprint, and can include additional resources and patches.

## How to deploy
The whole deployment workflow is based on knictl CLI tool that this repository is providing.

 **1. Fetch requirements for a site.**
You need to have a site repository with the structure described above. Then, first thing is to fetch the requirements needed for the blueprint that the site references. This is achieved by:

    ./knictl fetch_requirements  github.com/site-repo.git
Where the first argument references a site repository, following [go-getter](https://github.com/hashicorp/go-getter) syntax.
This will download the site repository, and will create a folder with the site name inside $HOME/.kni . It will also fetch all the binaries needed, and will store them inside $HOME/.kni/\$SITE_NAME/requirements folder.

 **2. Prepare manifests for a site**
Next step is to run a procedure to prepare all the manifests for deploying a site. This is achieved by applying kustomize on the site repository, combining that with the base manifests for the blueprint, and doing a merge with the manifests generated by the installer at runtime. This is achieved by the following command:

     ./knictl prepare_manifests $SITE_NAME
This will generate a set of manifests ready to apply, and will be stored on $HOME/.kni/\$SITE_NAME/final_manifests folder.
Along with manifests, a **profile.env** file has been created also in $HOME/.kni/\$SITE_NAME folder. It includes environment vars that can be sourced before deploying the cluster. Current vars that can be exported are:

 - OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE : used when a new image is wanted, instead of the default one
 - TF_VAR_libvirt_master_memory, TF_VAR_libvirt_master_vcpu. Used in the libvirt case, to define the memory and CPU for the vms.

 **3. Deploy the cluster**
Before starting the deployment, it is recommended to source the env vars from profile.env . You can achieve it with:

    source $HOME/.kni/\$SITE_NAME/profile.env
Then, you need to deploy the cluster using the generated manifests. This can be achieved with:

    $HOME/.kni/$SITE_NAME/requirements/openshift-install create cluster --dir=$HOME/.kni/$SITE_NAME/final_manifests

This will deploy a cluster based on the specified manifests. You can learn more about how to manage cluster deployment and how to interact with it on [https://docs.openshift.com/container-platform/4.1/welcome/index.html](https://docs.openshift.com/container-platform/4.1/welcome/index.html)

In the case of baremetal, ignition files need to be applied to each machine, instead of running the create cluster command. You can prepare the ignition files running this command:

    $HOME/.kni/$SITE_NAME/requirements/openshift-install create ignition-configs --dir=$HOME/.kni/$SITE_NAME/final_manifests

Then copy the ignition files to each machine, according to your provisioning tool.

   **4. Apply workloads**
After the cluster has been generated, the extra workloads that have been specified in manifests (like kubevirt), need to be applied. This can be achieved by:

     ./knictl apply_workloads $SITE_NAME

This will execute kustomize on the site manifests and will apply the output to the cluster.
After that, the site deployment can be considered as finished.

   **5. Destroy site**
When needed, the site can be destroyed with the openshift-install command, using the following syntax:

    $HOME/.kni/\$SITE_NAME/requirements/openshift-install destroy cluster --dir $HOME/.kni/\$SITE_NAME/final_manifests
