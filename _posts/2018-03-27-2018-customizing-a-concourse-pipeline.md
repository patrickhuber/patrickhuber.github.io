# Customizing a Concourse Pipeline

## Overview

When consuming a concourse pipeline like [pivotal-cf/pcf-pipelines](https://github.com/pivotal-cf/pcf-pipelines) you will inevitably need to customize based on your configuration needs. 

In BOSH we have the ability us use ops files built into the cli, but in fly we must use external tools to enable the same behavior. 

If you go into the [pivotal-cf/pcf-pipelines customizing-the-pipelines](https://github.com/pivotal-cf/pcf-pipelines#customizing-the-pipelines) repo's README.md you will see the callout for customizing the pipelines with [yaml_patch](https://github.com/pivotal-cf/yaml-patch). We can also use the [bosh cli](https://github.com/cloudfoundry/bosh-cli/releases) to perform patching operations with the `bosh int` command. Because we most likely already have the bosh command I'll be using that to patch pipelines.

## Principles

Lets go over a few principles that will aid us when customizing a pipeline. 

1. <a name="modification-principle">**Modification Principle**</a> - Avoid modifying the pipeline source repo. This allows you to easily upgrade your customized pipeline because you can avoid doing complex merges each time the dependency changes.
2. <a name="dependency-principle">**Dependency Principle**</a> - Pin to a particular change set of your dependency and avoid floating on master. This will allow your customizations to be stable and will avoid breaking changes from being introduced until you have properly tested them.
3. <a name="secrets-principle">**Secrets Principle**</a> - Avoid storing secrets in pipelines. Instead use a config server to store secrets.
4. <a name="portability-principle">**Portability Principle**</a> - Avoid storing environment specific configuration in your pipeline and instead export this configuration to environment variables or vars files. This allows the same pipeline to be used across multiple instances of concourse.

## Architecture

Based on the principles above, the following architecture decisions are outlined along with the principles that they implement.

### Submodules

Git submodules allow the customized pipeline repo to maintain a clear, source controlled upgrade path. The submodule directory keeps the dependency repo in a distinct location from the customizations so when an update occurs to the dependency, it is easily consumed by the customization repo. 

Creating a submodule is pretty straight forward. From within the customized repo you can consume a dependency using the `git submodule` command.

For this example, I will use a very simple concourse pipeline in [starkandwayne/concourse-tutorial](https://github.com/starkandwayne/concourse-tutorial) as my dependency. To setup this dependency as a submodule, I will type:

```bash
git submodule add https://github.com/starkandwayne/concourse-tutorial submodules/starkandwayne/concourse-tutorial/
```

I now have the submodule/starkandwayne/concourse-tutorial directory that contains the dependency for my repo.

### Config Server

In some example repos, a yaml file is included that contains variables and some of these variables contain secrets. To adhere to our secrets principle, we must never check these secrets into source control. The config server allows us to exclude secrets from our pipelines and they are looked up by concourse when it accesses the config server. 

Two config servers are supported by concourse:

1. CredHub
2. Vault

CredHub and Vault can both be deployed as standalone or alongside the bosh director. For stability, security and reliability it is more desirable to deploy them as standalone. For simplicity we will assume that our config server is available at a single IP address and that we are using the bosh deployment of concourse to configure the config server. 

#### CredHub 

##### Concourse Server Configuration

For CredHub the following ops file can enable credhub support. This was tested on concourse 3.8.

```yaml
- type: replace
  path: /instance_groups/name=web/jobs/name=atc/properties/credhub?
  value:
    client_id: ((credhub_client_id))
    client_secret: ((credhub_client_secret))
    path_prefix: /concourse
    tls:
      insecure_skip_verify: true
      ca_cert: ((credhub_ca_cert))
      url: ((credhub_url))
```

In version 3.8 of Concourse, the `insecure_skip_verify` property is required to be `true` due to a bug in the way certifictes are managed. In future version 3.9.2 this is fixed and this flag can be set to false. 

The `ca_cert` is the public certificate for the credhub server used to ensure TLS communications are secure. The URL is the url to the credhub instance or a load balanced url to the credhub url. 

The `path_prefix` property is important if you have multiple instances of concourse accessing the same CredHub instance. You will want to have distinct paths for all secrets and making this unique per CredHub instance keeps concourse instances from overwriting eachother's secrets. 

The `credhub_client_id` and `credhub_client_secret` are generated as part of your CredHub install. 

##### CredHub Cli

Download and install the [credhub cli](https://github.com/cloudfoundry-incubator/credhub-cli/releases) for easy management of client secrets. You can not write secrets into CredHub from the fly cli, so the secrets need to be in place before running `fly set-pipeline`.

The following examples show how to write various secrets. It is important to note that secrets are partitioned at team and pipeline levels. Adjusting the pattern will allow the secret to be read across pipelines in the team or isolated to a single pipeline. The [credential lookup rules](http://concourse-ci.org/creds.html#credential-lookup-rules) show the how credentials are written, so the pattern must be used when writing secrets in order for concourse to find them in CredHub. 

For a comprehensive example set using the cli and api see the [CredHub documentation](https://credhub-api.cfapps.io/#set-credentials).

###### Value Secret at Pipeline Scope

>command

```bash
# path prefix = /concourse, team = main, pipeline = test-pipeline
credhub set --name=/concourse/main/test-pipeline/some-value --type=value --value=abc123
```

>response

```yaml
id: d93c3289-ab31-4ac6-bb9a-f7bb79697a4d
name: /concourse/main/test-pipeline/some-value
type: value
value: abc123
version_created_at: 2018-03-27T19:16:22Z
```

###### Value Secret at Team Scope

```bash
# path prefix = /concourse, team = main
credhub set --name=/concourse/main/team-value --type=value --value=foobar
```

>response

```yaml
id: 2677aaa1-ca92-4e86-8c45-c8fc33783021
name: /concourse/main/team-value
type: value
value: foobar
version_created_at: 2018-03-29T14:21:36Z
```

###### Password Secret at Pipeline Scope

>command

```bash
# path prefix = /concourse, team = main, pipeline = test-pipeline
credhub set --type password --name '/concourse/main/test-pipeline/postgresql_password' --password '3t6Y2OFP0jQIcLnki1h7p3NtSfDx4l9bamr1ja6R'
```

>response

```yaml
id: c43e9da7-07bd-46ba-83e7-766e2d7cbe63
name: /concourse/main/test-pipeline/postgresql_password
type: password
value: 3t6Y2OFP0jQIcLnki1h7p3NtSfDx4l9bamr1ja6R
version_created_at: 2018-03-29T14:36:03Z
```

###### Certificate Secret at Team Scope using Command Line and creds yaml file from bosh

>command

```bash
# assume creds.yml is in the current directory. This file is generated by the credhub.yml ops file https://github.com/cloudfoundry/bosh-deployment/credhub.yml
# we use bash process substition to make the bosh int commands appear as files so we don't need to break things out https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html
# bosh environment name = bosh-1, concourse deployment name = concourse, variable name = credhub_ca
credhub set --type certificate --name '/bosh-1/concourse/credhub_ca' \
  --root <(bosh int creds.yml --path /credhub_ca/ca ) \
  --certificate <(bosh int creds.yml --path /credhub_ca/certificate ) \
  --private <(bosh int creds.yml --path /credhub_ca/private_key ) 
```

>response

```yaml
id: 159fa9b6-45e2-4df2-9960-78e06be6f3e7
name: /bosh-1/concourse/credhub_ca
type: certificate
value:
  ca: |+
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  certificate: |+
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  private_key: |+
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----

version_created_at: 2018-03-29T15:19:09Z
```

#### Vault

##### Concourse Server Configuration

* TODO: Create Vault Concourse Configuration

##### Vault Cli

* TODO: Create Vault Cli Configuration

### Encryption

The encrpytion flag in concourse encryptes any variables that are stored in concourse at the database level, but when these secrets are read by the `fly get-pipeline` command, they are decrypted and shown in plain text on the terminal. For this reason, we can not use encryption alone in concourse and must also use a config server. With a config server setup, the credentials will be masked and shown as their original variable names in double paranthesis `((variable_example))`.  

With a bosh deployment, the encryption can be setup using an [ops file](https://github.com/concourse/concourse-bosh-deployment/blob/master/cluster/operations/encryption.yml). The credential needs to be either 16 or 32 ASCII characters (16 or 32 bytes). For this reason it is best to pre generate the encryption key and store it in CredHub or Vault rather than rely on the auto generation feature that comes with BOSH. 

Using the CredHub CLI this can be done with the `credhub generate` command.

> command 

```bash
# bosh environment name = bosh-1, deployment name = concourse, encryption key variable = encryption_key
credhub generate --name=/bosh-1/concourse/encryption_key --type=password --include-special --length=16

# or 32 bytes
credhub generate --name=/bosh-1/concourse/encryption_key --type=password --include-special --length=16
```

> response

```yaml
id: c54a7387-ac93-4e5e-a76f-6c8480e58460
name: /bosh-1/concourse/encryption_key
type: password
value: 1%Ta1RzTzsOUa//L
version_created_at: 2018-03-29T17:59:26Z
```

Assuming the deployment name and environment name match the path, bosh will pick up the encryption password from CredHub when deploying concourse.

### Shell Scripts



### Operations Files

### Vars Files