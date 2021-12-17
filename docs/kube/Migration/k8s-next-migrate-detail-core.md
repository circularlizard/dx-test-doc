---
id: k8s-next-migrate-detail-core
title: Operator migration - core
---

This section will documented a overview how it is possible to migrate a core profile from a operator based deployment to a HELM based deployment. Also a output of this design is a first draft for the raw documentation.

## Migration-Mode for the HELM deployment

Before we can start to document the backup and restore/migration of the core profile we should implemented a migration mode. The goal of this mode is to start the core pod but without starting DX. The Pod should stay alive without DX runninng, so it is possible to connect to the pod and perform actions on the file system.

For the migration mode need to should adjust the following areas:

- Adding a migration parameter to the HELM chart
- Liveness, readiness, startUp - checks
- Core start scripts

Here are some example implementations on how to do that, as I have tested this in feature branches.

- https://git.cwp.pnp-hcl.com/websphere-portal-incubator/dx-helm-charts/tree/feature/DXQ-19119
- https://git.cwp.pnp-hcl.com/Team-Q/Portal-Docker-Images/compare/feature/DXQ-19119

The migration mode should be also interested for DAM. Here are the needed changes.

HELM charts
```
add config map entry

"dam.config.dam.migration": "{{ .Values.migration.enabled }}"
```

```
# Liveness probe
{{- if .Values.migration.enabled }}
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - exit 0
  failureThreshold: {{ .failureThreshold }}
  initialDelaySeconds: {{ .initialDelaySeconds }}
  periodSeconds: {{ .periodSeconds }}
  successThreshold: {{ .successThreshold }}
  timeoutSeconds: {{ .timeoutSeconds }}
{{ else }}
livenessProbe:
{{- with .Values.probes.digitalAssetManagement.livenessProbe }}
  {{- toYaml . | nindent 12 }}
{{- end }}
{{- end }}
# Readiness probe
{{- if .Values.migration.enabled }}
readinessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - exit 0
  failureThreshold: {{ .failureThreshold }}
  initialDelaySeconds: {{ .initialDelaySeconds }}
  periodSeconds: {{ .periodSeconds }}
  successThreshold: {{ .successThreshold }}
  timeoutSeconds: {{ .timeoutSeconds }}
{{ else }}
readinessProbe:
{{- with .Values.probes.digitalAssetManagement.readinessProbe }}
  {{- toYaml . | nindent 12 }}
{{- end }}
{{- end }}
```

Start scripts:
```
#!/bin/bash

# Uses environment variables from a config map if mounted
for file in /etc/config/dam.config.dam.*; do
  # Loop through all the files in directory and check if file exists.
  if test -f "$file"; then
    # Split the filename into array with dot(.) delimiter.
    IFS='.' read -r -a array <<< "$(basename "$file")"
    # Get the last element of array for eg. postgres_db.
    envVariableTemp=${array[@]: -1:1}
    # Capitilise the string.
    envVariable=$(tr '[a-z]' '[A-Z]' <<< $envVariableTemp)
    fileName="${file##*/}"
    # Assign the variables with contents of file. eg. POSTGRES_DB='testdb'.
    printf -v $envVariable $(cat /etc/config/$fileName)
    # export the variable as environment variable.
    export $envVariable
  fi
done

clean_up () {
  echo "Sending SIGTERM to all node instances..."
  cd /opt/app/server-v1 && npm run docker:stop-daemon
}

stop_container () {
  echo "Caught SIGTERM, stopping DX Core container"
  exit 0
}

echo "Migration is: $MIGRATION"

if [ "$MIGRATION" == "true" ]; then
  trap stop_container SIGTERM
  echo "Listening for SIGTERM"
  echo "Migration is enabled, stay alive and do nothing ..."
  # ensure the docker container stays alive and can listen for signals every second
  while true; do sleep 1; done
else
  trap clean_up SIGHUP SIGINT SIGTERM

  # run server version 1 
  cd /opt/app/server-v1 && npm run docker:start
fi
```

## Backup and restore process (raw doc)

This section documents the migration process of the core profile.

### Backup form the operator based deployment

#### 1. Ensure that only one core pod is running

Check how many pods are running. Use the following command for that.

```
kubectl -n <namespace> get all
```

If more then one core pod running, scaling down the core pods to only one. On the operator deployment adjust the DXCTL property file and apply this changes via the DXCTL tool.

#### 2. Connecting to the core pod

With the following command it is possible to jump directly into the running container.

```
kubectl exec --stdin --tty pod/<pod-name> -n <namespace> -- /bin/bash
```

Example:
```
kubectl exec --stdin --tty pod/dx-deployment-0 -n dxns -- /bin/bash
```

#### 3. Stop the server

Before a backup from the wp-profile can be create, the core application should be stopped. 

Navigate to the profile `bin` folder and run the `stopServer` command.

```
cd /opt/HCL/wp_profile/bin/

./stopServer.sh WebSphere_Portal -username <username> -password <password>
```

#### 4. Compress the profile

For the operator-based deployment is the whole core profile located under `/opt/HCL/wp_profile`. This folder must be compressed.

```
cd /opt/HCL
tar -cvpzf core_prof_95_CF197.tar.gz --exclude=/core_prof_95_CF197.tar.gz --one-file-system wp_profile
```

#### 5. Download the backup profile

From a local system it is now possible to download the backup core profile from the core pod container.

```
kubectl cp <namespace>/<pod-name>:<source-file> <target-file>
```

Example:
```
kubectl cp dxns/dx-deployment-0:opt/HCL/core_prof_95_CF197.tar.gz /tmp/core_prof_95_CF197.tar.gz 
```

### Restore to the HELM based deployment

#### 1. Start the HELM deployment

Before we can start with the restore we should ensure that the HELM based deployment is in correct state. Some adjustments are necessary.

- The extraction of kubernetes DX configuration from the operator based deployment to a valid `values.yaml` file is done.
- Enable the `migration` mode.
- For the first start the `runtimeController` and the `core` application should be enabled.

Now the HELM deployment can be started.

```
helm install -n <namespace> --create-namespace -f <values.yaml> <prefix> <chart>
```

Example:
```
helm install -n dxns --create-namespace -f hcl-dx-deployment/value-samples/internal-repos.yaml hha hcl-dx-deployment
```

What we expected now is:

- The `core` pod is running and keep alive.
- The `core` application is NOT running.
- No default profile should be created automatically. The folder `/opt/HCL/wp_profile` is empty.

#### 2. Upload the backup profile

Now it is possible to transfer the backup profile to the remote core pod container.

```
kubectl cp <source-file> <namespace>/<pod-name>:<target-file>
```

Example:
```
kubectl cp /tmp/core_prof_95_CF197.tar.gz dxns/hha-core-0:/tmp/core_prof_95_CF197.tar.gz
```

#### 3. Connect to the core pod

With the following command it is possible to connect directly into the running container.

```
kubectl exec --stdin --tty pod/<pod-name> -n <namespace> -- /bin/bash
```

Example:
```
kubectl exec --stdin --tty pod/hha-core-0 -n dxns -- /bin/bash
```

#### 4. Extracting the profile

At first the backup file `core_prof_95_CF197.tar.gz` should moved from `/tmp` folder to the profile folder `/opt/HCL/profiles`. And the extraction can be started.

```
tar -xf /tmp/core_prof_95_CF197.tar.gz --directory /opt/HCL/profiles
mv /opt/HCL/profiles/wp_profile /opt/HCL/profiles/prof_95_CF197
rm /tmp/core_prof_95_CF197.tar.gz
```

The next step is to create a symlink.

```
rm -r /opt/HCL/wp_profile
ln -s /opt/HCL/profiles/prof_95_CF197 /opt/HCL/wp_profile
```

#### 5. Disable the migration mode and the deployment

Before we can start with the final upgrad of the HELM deployment some adjustments are necessary.

- Disable the `migration` mode.
- Enable all relevant applications.

Now the HELM deployment can be upgraded.

```
helm upgrade -n <namespace> --create-namespace -f <values.yaml> <prefix> <chart>
```

Example:
```
helm upgrade -n dxns --create-namespace -f hcl-dx-deployment/value-samples/internal-repos.yaml hha hcl-dx-deployment
```
