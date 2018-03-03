# Customizing a Bosh Deployment

## Overview 

I've been spending a lot of time with [BOSH](https://bosh.io) recently and a lot of the work revolves around creating customized deployments. 

Most projects you work with will have a deployment manifest that you need to customize through ops files. You will want to store your customizations in a repo that you control becuase your configuration will almost always be different than the configuration provided by default. The perfect example of this is the number of AZs. 

## Principles

Lets go over a few principles that we want to establish which will guide the workflow we adopt.

1. <a name="modification-principle">**Modification Principle**</a> - Avoid modifing deployment repo dependencies. Modifying dependencies can result in a brittle deployment. If the dependency updates and you have made modifications you will need to track those modifications with merges which is a lot of overhead over just getting the latest version and re-deploying. 
2. <a name="dependency-principle">**Dependency Principle**</a> - Take a hard dependency on a version/release of a deployment repo. In order to make your deployment more resilient, fix your dependencies to a specific version. This can take the form of a specific URI or tags/commit identifiers in source control.
3. <a name="secrets-principle">**Secrets Principle**</a> - Don't store secrets in source control, always use a configuration server like CredHub. The best way get breached or to be exploited by farming bots is to put your secrets in source control (so don't do it). Using a config server eliminates the risk of accidently checking in a secret vars file or forgetting to mask a directory with .gitignore. Not to mention it prevents secrets from being stored in plain text on a file system.
4. <a name="portability-principle">**Portability Principle**</a> - In order to remain portable, try to push configuration that varies between deployments to environment specific configuration. This is a customization of a deployment so this includes structural things like number of AZs and instance counts. Having a portable configuration will make it easier to move to a different environment. In general, if something changes between deployment environments, it is a good candidate for pulling out into environment specific configuration.

## Architecture

Lets address each princple with a decision and then show how these decisions can be supported with a workflow, directory structure and source control features.

### Git Submodules

In order to follow the [modification principle](#modification-principle) and [dependency principle](#dependency-principle) we can use git submodules. Submodules allow us to take a snapshot of a deployment repo and track the version information of that snapshot in source control along side our deployment customizations. 

The commands for adding a submodule are pretty simple

```bash
git submodule add https://github.com/cloudfoundry-incubator/kubo-deployment submodules/github.com/cloudfoundry-incubator
```

The key is to pin the submodule to a particular commit or tag. Often deployment authors will create releases of their deployments and tag the releases. This creates a reference commit and associates a tag with the reference commit. If a release is not specified, you can just pin to the commit on master at the time you are creating your customized deployment.

> by commit hash

```bash
cd submodules/github.com/cloudfoundry-incubator
git checout a291edc
```

> or by tag

```bash
cd submodules/github.com/cloudfoundry-incubator
git checkout v0.13.0
```

For all intensive purposes, this is a version identifier that pins your deplendency which is great. The last thing we want is to take a dependency on the ever changing latest commit on master and get out of sync with the deployment development team. 

### Configuration Server

The BOSH Configuration Server is key to keeping the customized repo clean of secrets and specific configuration items that limit its portability. This will allow us to implement the [secrets principle](#secrets-principle). The configuration server should be kept clear of structural things like number of AZs or instance counts. Those elements are good candidates for environment specific configuration which adhears to the portability principle. Storing secrets in the config server is not only safe and protects you from secret leaks, it also has the added benefit of keeping our customized deployment clean of gitignore masked directories.

CredHub is the defualt configuration server supplied through the https://github.com/cloudfoundry/bosh-deployment/credhub.yml ops file. You simply need to add the uaa.yml and credhub.yml as ops files to your deployment of bosh in order to set it up. [This repo](https://github.com/nsagoo-pivotal/concourse-credhub-bosh-deployment) does an excellent job of stepping through the setup. 

You will still need to deal with the zero secrets problem, luckily password managers like lastpass and onepassword have CLIs that can be used to pull the initial secrets needed to access CredHub.

### Shell Scripts

If we follow the principles and architecture decisions above, we can create shell scripts that farm out our deployment information to a single command. With a shell script, we no longer need to remember a lengthy command and, as long as we utilize a config server, use path variables instead of hard coding values. 

Secrets can be specified with the [credhub cli](https://github.com/cloudfoundry-incubator/credhub-cli/releases) out of band of the deployment script. With secrets securly in credhub, we only need to reference the secret by path in our deployment manifest. 

Here is a sample deployment script for deploying kubo (aka cfcr) using the techniques above:

> [deploy-kubo.sh](https://github.com/patrickhuber/vsphere-kubo-deployment/blob/master/deploy-kubo.sh)

```bash
bosh deploy -d kubo submodules/github.com/cloudfoundry-incubator/kubo-deployment/manifests/cfcr.yml \
  -o submodules/github.com/cloudfoundry-incubator/kubo-deployment/manifests/ops-files/vm-types.yml \
  -o submodules/github.com/cloudfoundry-incubator/kubo-deployment/manifests/ops-files/iaas/vsphere/cloud-provider.yml \
  -o submodules/github.com/cloudfoundry-incubator/kubo-deployment/manifests/ops-files/iaas/vsphere/master-static-ip.yml \
  -o submodules/github.com/cloudfoundry-incubator/kubo-deployment/manifests/ops-files/iaas/vsphere/set-working-dir-no-rp.yml \
  -o ops-files/one-az.yml \
  -o ops-files/deployment-name.yml \
  -o ops-files/remote-kubo-release.yml \
  -v master_vm_type=default \
  -v worker_vm_type=default \
  -v deployment_name=kubo \
  -v kubernetes_master_host=192.168.2.20
```

As a general practice I put external dependencies above specific dependencies of my deployment so that my deployment settings override the settings of the dependency (which is often the indended behavior).

You'll see I have the master IP address coded in this shell script, I would argue this violates the [portability principle](#portability-principle) as it ties me to a specific network configuration. I did it for simplicity when setting up this deployment and will move it into the configuration server.

### Ops Files

[Ops files](https://bosh.io/docs/cli-ops-files.html) are the key to customizing a deployment. Most deployment place ops files in some distinct directory like "ops-files" or "ops", I recommend doing the same as it keeps the root directory clear of your modifications and makes it clear at first glance how to execute the deployment. Using ops files allows us to adhear to the [modification principle](#modification-principle) by not mutating our dependency.

You can see in the shell scripts section above that I used custom ops files to:

1. create a single availiblity zone
2. change the deployment name
3. update the location of the kubo release artifacts (I prefer to have BOSH check my dependencies).

The updating the release artifact location helps to implement the [dependencies principle](#dependencies-principle). We want to know the exact version of the release we have a dependency on. Leaving this value floating can lead to a broken deployment if another deployment happens to use the same release but of a different version. 

In general I try to make ops files declarative in name and fulfill a specific purpose. For example 'one-az.yml' is very clear that the ops file sets the AZ count to 1. If you need more flexibilty, you can add an ops file that creates a variable. Do this if the value needs to be updated regularly or is specified through another variable that you want to reference. 

### Defaults

#### Cloud Config

I think it is a good idea to specify defaults in your cloud config. Example deployments often specify default VM types with the name 'default' and this can greatly speed up the time to your first successful deployment. In general, you should specify a default:

1. vm_type
2. network
3. disk_type

#### Environment

I like to specify a default environment using the $BOSH_ENVIRONMENT variable. If you are using more than one bosh environment from the same client, I'd suggest instead pushing the environment selection to the -e flag of the bosh cli. This will avoid the accidental overwriting of the wrong environment.

## Workflow

Now that we have an archtiecture base, lets dive into the workflow.

### Create Initial Repo

The start of the process is to create a repo to store your customizations. Typically you create a directory, change the current directory into the directory you just created and then type 'git init'. Lets say we are creating a deployment for Vault:

```bash
mkdir vault-deployment-vsphere
cd vault-deployment-vsphere
git init
```

You can name the directory anything you like. Typically the dependencies will be named [something]-deployment so I try to make the name distinct to avoid confusion.

### Create Dependency Submodules

As we showed above, we then need to establish our dependencies. For the vault release, we need to take a dependency on the [vault deployment](https://github.com/cloudfoundry-community/vault-boshrelease). Sometimes a distinct repo exists for a deployment, and sometimes the deployment and release are coupled in the same repo. This is an example of the release and sample deployment being coupled.

To establish the dependency, we use the submodules command from above to create a dependency and then use the checkout command within the submodule directory to pin it to a specific version. 

```bash
git submodule add https://github.com/cloudfoundry-community/vault-boshrelease/ submodules/github.com/cloudfoundry-community
```

```bash
cd submodules/github.com/cloudfoundry-community
git checkout v0.7.0
```

I like to organize submodules by (similar to how golang organizes dependencies), though you can use whatever scheme works for you:

1. source control provider
2. org or user
3. repo

### Work Loop

#### Shell Scripts 

As we are working on the shell script to customize our deployment, we will most likely go through a modification loop between updating the shell script, writing secrets to the config server and creating ops files.

To get started, lets create a shell script for deploying vault. I'll call it 'deploy-vault.sh'.

```bash
bosh deploy -d vault \
     submodules/cloudfoundry-community/vault-boshrelease/manifests/vault.yml
```

Next modify it for execute

```bash
chmod u+x depoy-vault.sh
```

Fire it off :). Something I learned from a dojo is to fail as soon as possible so you can start your iteration loop. No fear, bosh is here.

```bash
./deploy-vault.sh
```

The vault release is very simple and as long as we specify defaults, we won't need to bring other ops files in or variables. The next section will cover a more complicated deployment configuration.

Don't forget to save your changes and push everything to the remote repo!

```bash
git add -A
git commit -m "created a customized bosh deployment for vault"
# add your remote here if you haven't done so
git push origin master
```

#### Ops Files

Now I want to modify the deployment to change the number of AZs from 1 to 3. The vault bosh release comes with a ops file for modifying the AZ list [here](https://github.com/cloudfoundry-community/vault-boshrelease/blob/master/manifests/operators/azs.yml). This allows me to specify a different number of AZs for each of my environments. This is great if I have say a lab environment with only one AZ and a QA or Production environment with the recommended 3.

Update the deploy-vault.sh file with the custom ops file

```bash
bosh deploy -d vault \
     submodules/cloudfoundry-community/vault-boshrelease/manifests/vault.yml \
     -o submodules/cloudfoundry-community/vault-boshrelease/manifests/operators/azs.yml \
     -v azs=[az1,az2,az3]
```

Deploy changes:

```bash
./deploy-vault.sh
```

Save changes:

```bash
git add -A
git commit -m "created a customized bosh deployment for vault"
# add your remote here if you haven't done so
git push origin master
```

### Updating the deployment

Now that I have a sample deployment of relative complexity, I am going to update the deployment from v0.7.0 to v0.8.0. This is relatively easy because I'm using a submodule that is tracked by source control. 

First things first, I'll change the working directory to the submodule directory

```bash
cd submodules/cloudfoundry-community/vault-boshrelease
```

Next I need to get the tagged version of the repo

```bash
git checkout v0.8.0
```

Next I need to deploy

```bash
cd ../../..
./deploy-vault.sh
```

Now that my vault deployment is completed, I'm going to save my changes:

```bash
git add -A
git commit -m "created a customized bosh deployment for vault"
# add your remote here if you haven't done so
git push origin master
```

### Environment Specific Vars Files

I want to keep my deployment portable so I'm going to write the az configuration to distinct files that mirror my environment structures. I'm then going to update the bash script with a configurable parameter with a default. This will allow me to keep the script simple and override the defaults in my production CI pipeline.

Like I mentioned before, I have a single AZ in my lab environment and have three AZs in my production environemnt. To represent those two environments I'm going to create environment specific vars files. 

```bash
mkdir vars-files
cat <<EOF > vars-files/lab.yml
---
azs: [z1]
EOF
cat <<EOF > vars-files/prod.yml
---
azs: [z1,z2,z3]
EOF
```