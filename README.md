# kubernetes-otc

Kubernetes OneTimeCommand Executor. Create ad-hoc Kubernetes jobs which can be peer-reviewed, approved and applied 
towards production environments. Allows review on actions in production environment, eg retrieve (limited) production 
data for debugging purposes. Can encrypt pipeline output to make sure production data remains sufficiently 
protected.

## How to use it

### with cassandra

1. replace `microservice_name` with your value in [service-cm](/overlays/service/service-cm.yaml)
2. add cqlsh query into [query](/overlays/type/cassandra/query.cql)
3. optional: for very specific use cases annotations must be modified [patch.yaml](/overlays/type/cassandra/patch.yaml)
4. change the date on which you want to execute the query in [service-cm.yaml](/overlays/service/service-cm.yaml)

tips: Remember that query should ends by ";". Also use table name with keyspace name e.g. "providers.client_authentication_means"

### with postgres

1. replace `microservice_name` with your values in [service-cm](/overlays/service/service-cm.yaml)
1. replace `database_name` with your values in [service-cm](/overlays/service/service-cm.yaml)
2. `rds_hostname_prefix` is mapped to [`RDSInstances[HostPrefix]`](https://git.yolt.io/infra/vault-cfg/-/blob/master/vault-prd.yml#L178) and `rds_instance_name` is mapped to [`RDSInstances[Name]`](https://git.yolt.io/infra/vault-cfg/-/blob/master/vault-prd.yml#L178)
3. add sql query into [query](/overlays/type/postgres/query.sql)
4. change the date on which you want to execute the query in [service-cm.yaml](/overlays/service/service-cm.yaml)
5. optional: for very specific use cases annotations must be modified [patch.yaml](/overlays/type/postgres/patch.yaml)

### with curl

1. replace `microservice_name` with your value in [service-cm](/overlays/service/service-cm.yaml)
2. modify [patch.yaml](/overlays/type/curl/patch.yaml) to include your shell script after `set -e` line in the following format:

```bash
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-curl.sh: $(ENVIRONMENT_NAME)/database/cassa/creds/$(NAMESPACE_PREFIX)$(MICROSERVICE_NAME)
        vault.hashicorp.com/agent-inject-template-curl.sh: |
      ### Uncomment and change accordingly follwing line if you need to use secrets from vault:
          # {{- with secret "$(ENVIRONMENT_NAME)/database/cassa/creds/$(NAMESPACE_PREFIX)$(MICROSERVICE_NAME)" -}}
          #!/bin/bash
          set -e;
      ### Insert your script here.
      ### the following line will run curl to get health of the service configured in step #1 of this section
          curl -s http://$(MICROSERVICE_NAME)/$(MICROSERVICE_NAME)/actuator/health
      ### Uncomment the follwing line if you need to use secrets from vault:
          # {{- end }}
```

3. make sure do not remove the date validation, only modify the date `run_only_on`

```bash
    #!/bin/bash
    set -e +x
    today=$(date +%d-%m-%Y)
    run_only_on="25-09-2020"
    if [ "$run_only_on" == "$(EXECUTION_DATE)" ];
    then
     curl -skX POST https://$(MICROSERVICE_NAME)/$(MICROSERVICE_NAME)/XXXXX
    fi
```

4. change the date on which you want to execute the query in [service-cm.yaml](/overlays/service/service-cm.yaml)

### with simple-curl

1. modify the curl in [job.yaml](/overlays/type/simple-curl/job.yaml)
2. change the date on which you want to execute the query in [service-cm.yaml](/overlays/service/service-cm.yaml)


## Data encryption (GPG)

Production job doesn't print output of the query. Instead it creates a file called `command-output.gpg` which uses GPG public key from users' gitlab profile to encrypt output (assume we have sensitive data there). Encrypted file is available as an artifact of the job. To decrypt the output of the command user must:

* download and extract artifact (artifact TTL - 1 hour)
* run `cat command-output.gpg | gpg --decrypt`

    on a MacOS you might get an error saying
    ```
    gpg: public key decryption failed: Inappropriate ioctl for device
    gpg: decryption failed: No secret key
    ```
    If that happens, execute `export GPG_TTY=$(tty)` before the above command. Consider putting it into your `.rc` (`.bashrc`, `.zhsrc` etc) file to make it    permanently applicable.

    An alternative syntax `gpg --decrypt command-output.gpg` should also work just fine. 

Key concept:

* user must [add gpg public key](https://docs.gitlab.com/ee/user/project/repository/gpg_signed_commits/#adding-a-gpg-key-to-your-account) to his profile
* encryption works only on `master` due to gitlab limitation for fetching public gpg key of user
* no use of vault due to its centralization - adds more headache on how to grant permissions to decrypt data + gives potential ability to decrypt all outputs of all previously executed jobs

## Sensitive arguments

Please do not include sensitive information unencrypted in the OTC queries as this is then part of source control.

## Debugging
1. Run it on ACC first. This can be done already on feature branch. If the execution is successful, the output is simply part of console output in GitLab.
2. In case the execution fails or times out, use `kubectl` to get more details about the error(s)
   1. `kubectl get jobs | grep otc` - display jobs created by kubernetes-otc
   2. `kubectl describe job kubernetes-otc-<jobname>` - display details of spcific job
   3. In the end of the output you can find names of the pod(s) created to execute job 
   4. To see logs use `kubectl logs -f <podname>`

### Cleanup
If the execution of the job has failed, it is not deleted automatically from k8s cluster.
Also the related pod keeps "hanging" in the error state.
Remove the job manually using `kubectl delete job kubernetes-otc-<jobname>`
   
