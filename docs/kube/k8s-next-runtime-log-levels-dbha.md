---
id: k8s-next-runtime-log-levels-dbha
title: Runtime log levels - DBHA
---

## Postgres

The log levels for Postgres are defined by the [`log_min_messages` parameter](https://www.postgresql.org/docs/9.1/runtime-config-logging.html#GUC-LOG-MIN-MESSAGES). Supported values are:

`DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `LOG`, `FATAL`, `PANIC`

Log levels for Postgres can be adjusted in different ways. 

### Update postgresql.conf 

In the current build of the DBHA Postgres image, the `postgresql.conf` is located in `/var/lib/pgsql/11/data/dx/`. It includes the `log_min_messages` which is commented out by default. After it was changed, the configuration can be reloaded without restarting Postgres with

```shell
repmgr node service --action=reload
```

### Update in database 

Another way of configuring the log level is to alter the entry in the database directly. For that we can connect to the database inside the container using `psql`. To change and apply the level change, we use

```sql
ALTER SYSTEM SET log_min_messages = 'DEBUG5';
SELECT pg_reload_conf();
```

## Repmgr

The log levels for the Repmgr are defined by the [`log_level` parameter](https://repmgr.org/docs/4.1/configuration-file-log-settings.html#REPMGR-CONF-LOG-LEVEL). Supported values are:

`DEBUG`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `ALERT`, `CRIT`, `EMERG`

Log levels for Postgres can be adjusted the following way. 

### Update repmgr.conf

In the current build of the DBHA Postgres image, the `repmgr.conf` is located in `/etc/repmgr/11`. It includes the `log_level` which is commented out by default. After it was changed, the configuration can be reloaded without restarting Repmgr with

```shell
kill -SIGHUP `cat /tmp/repmgrd.pid`
```

### Additional changes

In order to make those changes applicable, we need to make some slight changes to the `start_postgres.sh` script. We need to remove the following lines to enable logging to `stdout`:

```shell
__repmgr_set_property "log_file" "${PG_CONFDIR}/repmgr/log/repmgr.log" "${REPMGR_CONF_DIR}/repmgr.conf"
__repmgr_set_property "log_level" "NOTICE" "${REPMGR_CONF_DIR}/repmgr.conf"
```

### Periodically check for global log config and translate to compatible log config

We can create a shell script that runs in the image and periodically checks for changes in the global log config that is mounted in the container at `/etc/global-config/log.persistenceNode`.

Example log string:
```
pers-db:::info,pers-repmgr:::info
```

```shell
#!/bin/bash

#
####################################################################
# Licensed Materials - Property of HCL                             #
#                                                                  #
# Copyright HCL Technologies Ltd. 2021. All Rights Reserved.       #
#                                                                  #
# Note to US Government Users Restricted Rights:                   #
#                                                                  #
# Use, duplication or disclosure restricted by GSA ADP Schedule    #
####################################################################
#


# Load libraries
. /set_repmgr_property.sh

# Define service name
PG_SERVICE="pers-db"
REPMGR_SERVICE="pers-repmgr"
REPMGR_CONF_DIR='/etc/repmgr/11'

# Define log level mapping for Postgres
declare -A PG_MAP
PG_MAP['debug']='DEBUG5' # logs DEBUG5, DEBUG4, DEBUG3, DEBUG2, DEBUG1
PG_MAP['info']='INFO' # logs INFO, NOTICE, WARNING
PG_MAP['error']='ERROR' # logs ERROR, LOG, FATAL, PANIC

# Define log level mapping for Repmgr
declare -A REPMGR_MAP
REPMGR_MAP['debug']='DEBUG' # logs DEBUG
REPMGR_MAP['info']='INFO' # logs INFO, NOTICE, WARNING
REPMGR_MAP['error']='ERROR' # logs ERROR, ALERT, CRIT, EMERG


Map_Level()
{
  # Transform to lowercase
  INPUT_LEVEL=$(echo "$1" | tr '[:upper:]' '[:lower:]')

  SERVICE=$2

  # Returns mapped value if it exists, oterwise empty string
  if [[ $SERVICE == $PG_SERVICE ]]; then
    echo ${PG_MAP[$INPUT_LEVEL]}
  elif [[ $SERVICE == $REPMGR_SERVICE ]]; then
    echo ${REPMGR_MAP[$INPUT_LEVEL]}
  else
    echo ""
  fi
}

Parse_And_Set_Level()
{

  # Split by comma
  IFS=',' read -ra LOG_STR <<< $1

  for LOG_PART in "${LOG_STR[@]}"; do
    # Split by colon
    IFS=':' read -ra LOG_STR_PARTS <<< $LOG_PART

    # Get common log pattern elements
    SERVICE=${LOG_STR_PARTS[0]}
    # Transform to lowercase
    LOWERCASE_SERVICE=$(echo "$SERVICE" | tr '[:upper:]' '[:lower:]')

    # Suffix and component are currently not used in this service
    # SUFFIX=${LOG_STR_PARTS[1]}
    # COMPONENT=${LOG_STR_PARTS[2]}

    LEVEL=${LOG_STR_PARTS[${#LOG_STR_PARTS[@]} - 1]}

    # Only apply logs for the matching service
    if [[ $LOWERCASE_SERVICE == $PG_SERVICE ]]; then
      MAPPED_LEVEL=$(Map_Level "$LEVEL" $PG_SERVICE)

      if [[ $MAPPED_LEVEL != "" ]]; then
        # Setting the log level
        echo "Setting Postgres log level to $MAPPED_LEVEL" && \
        psql -c "ALTER SYSTEM SET log_min_messages = '$MAPPED_LEVEL';" && \
        psql -c "SELECT pg_reload_conf();" || \
        echo "Failed to set Postgres log level to $MAPPED_LEVEL"
      fi
    elif [[ $LOWERCASE_SERVICE == $REPMGR_SERVICE ]]; then
      MAPPED_LEVEL=$(Map_Level "$LEVEL" $REPMGR_SERVICE)

      if [[ $MAPPED_LEVEL != "" ]]; then
        # Setting the log level
        echo "Setting Repmgr log level to $MAPPED_LEVEL" && \
        __repmgr_set_property "log_level" "${MAPPED_LEVEL}" "${REPMGR_CONF_DIR}/repmgr.conf" && \
        kill -HUP `cat /tmp/repmgrd.pid` || \
        echo "Failed to set Repmgr log level to $MAPPED_LEVEL"
      fi
    fi

  done
}

Periodically_Read_Ronfigfile() {
  FILE_PATH=$1
  # Sleep time defaults to 10 seconds if not passed as argument
  SLEEP_TIME_SECONDS=${2:-10}

  LAST_CONFIG_STRING=''

  while [ true ]; do
    if [[ -f ${FILE_PATH} || -L ${FILE_PATH} ]]; then
      # Deal with the fact that the file referenced could actually be a symlink to a file
      FULLY_RESOLVED_FILE_PATH=$(readlink -f ${FILE_PATH})

      # read exactly the first line from that file
      read -r first_line < ${FULLY_RESOLVED_FILE_PATH}
      if [ "${LAST_CONFIG_STRING}" != "${first_line}" ]; then
        LAST_CONFIG_STRING=${first_line}
        Parse_And_Set_Level "$first_line"
      else
        echo "Nothing done because log config didn't change in ${FULLY_RESOLVED_FILE_PATH} in last" ${SLEEP_TIME_SECONDS} "seconds"
    fi
    else
      echo "Config map ${FILE_PATH} doesn't exist. No action taken"
    fi
    sleep ${SLEEP_TIME_SECONDS}
  done
}

# Start periodic file check
Periodically_Read_Ronfigfile "/etc/global-config/log.persistenceNode" 10
```

## Pgpool

The log levels for Pgpool are defined by the [`log_min_messages` parameter](https://www.pgpool.net/docs/41/en/html/runtime-config-logging.html#GUC-LOG-MIN-MESSAGES). Supported values are:

`DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `LOG`, `FATAL`, `PANIC`

Log levels for Pgpool can be adjusted in different ways. 

### Update pgpool.conf

In the current build of the DBHA Pgpool image, the `pgpool.conf` is located in `/conf`. It includes the `log_min_messages` which is commented out by default. After it was changed, the configuration can be reloaded without restarting Postgres with

```shell
pgpool --config-file=/conf/pgpool.conf --hba-file=/conf/pool_hba.conf reload
```

We are currently using Pgpool 4.1. With 4.2, a new command [`pcp_reload_config` was introduced](https://www.pgpool.net/docs/42/en/html/pcp-reload-config.html). If we upgrade to 4.2 in the future, this can probably replace the command above.

### Update in database *(Do not use for our case)*

Another way of configuring the log level is to alter the entry in the database directly. For that we can connect to the database inside the container using

**Important Note:** 
> Do not use this for our use case! This can in some cases work unreliably if it is executed during failover or the setting is not present in all nodes. The config file is better to store the setting in a persistent way.


```shell
psql postgres://<PGPOOL_SR_CHECK_USER>:<PGPOOL_SR_CHECK_PASSWORD>@localhost/postgres
```

Example:

```shell
psql postgres://repdxuser:d1gitalRepExperience@localhost/postgres
```

To change and apply the level change, we use

```sql
PGPOOL SET log_min_messages = 'DEBUG5';
```

### Periodically check for global log config and translate to compatible log config

We can create a shell script that runs in the image and periodically checks for changes in the global log config that is mounted in the container at `/etc/global-config/log.persistenceConnectionPool`.

Example log string:
```
pers-pool:::info
```

```shell
#!/bin/bash

#
####################################################################
# Licensed Materials - Property of HCL                             #
#                                                                  #
# Copyright HCL Technologies Ltd. 2021. All Rights Reserved.       #
#                                                                  #
# Note to US Government Users Restricted Rights:                   #
#                                                                  #
# Use, duplication or disclosure restricted by GSA ADP Schedule    #
####################################################################
#


# Load libraries
. /scripts/libpgpool.sh

# Load Pgpool env. variables
eval "$(pgpool_env)"

# Define service name
CURRENT_SERVICE="pers-pool"

# Define log level mapping
declare -A map
map['debug']='DEBUG5' # logs DEBUG5, DEBUG4, DEBUG3, DEBUG2, DEBUG1
map['info']='INFO' # logs INFO, NOTICE, WARNING
map['error']='ERROR' # logs ERROR, LOG, FATAL, PANIC


Map_Level()
{
  # Transform to lowercase
  INPUT_LEVEL=$(echo "$1" | tr '[:upper:]' '[:lower:]')

  # Returns mapped value if it exists, oterwise empty string
  echo ${map[$INPUT_LEVEL]}
}

Parse_And_Set_Level()
{
  # Ex: "pool:::info"

  # Split by comma
  IFS=',' read -ra LOG_STR <<< $1


  for LOG_PART in "${LOG_STR[@]}"; do
    # Split by colon
    IFS=':' read -ra LOG_STR_PARTS <<< $LOG_PART

    # Get common log pattern elements
    SERVICE=${LOG_STR_PARTS[0]}
    # Transform to lowercase
    LOWERCASE_SERVICE=$(echo "$SERVICE" | tr '[:upper:]' '[:lower:]')

    # Suffix and component are currently not used in this service
    # SUFFIX=${LOG_STR_PARTS[1]}
    # COMPONENT=${LOG_STR_PARTS[2]}

    LEVEL=${LOG_STR_PARTS[${#LOG_STR_PARTS[@]} - 1]}

    # Only apply logs for the matching service
    if [[ $LOWERCASE_SERVICE == $CURRENT_SERVICE ]]; then
      MAPPED_LEVEL=$(Map_Level "$LEVEL")

      if [[ $MAPPED_LEVEL != "" ]]; then
        # Setting the log level
        echo "Setting log level to $MAPPED_LEVEL" && \
        pgpool_set_property "log_min_messages" $MAPPED_LEVEL && \
        pgpool --config-file=$PGPOOL_CONF_FILE --hba-file=$PGPOOL_PGHBA_FILE reload || \
        echo "Failed to set log level to $MAPPED_LEVEL"
      fi
    fi

  done
}

Periodically_Read_Ronfigfile() {
  FILE_PATH=$1
  # Sleep time defaults to 10 seconds if not passed as argument
  SLEEP_TIME_SECONDS=${2:-10}

  LAST_CONFIG_STRING=''

  while [ true ]; do
    if [[ -f ${FILE_PATH} || -L ${FILE_PATH} ]]; then
      # Deal with the fact that the file referenced could actually be a symlink to a file
      FULLY_RESOLVED_FILE_PATH=$(readlink -f ${FILE_PATH})

      # read exactly the first line from that file
      read -r first_line < ${FULLY_RESOLVED_FILE_PATH}
      if [ "${LAST_CONFIG_STRING}" != "${first_line}" ]; then
        LAST_CONFIG_STRING=${first_line}
        Parse_And_Set_Level "$first_line"
      else
        echo "Nothing done because log config didn't change in ${FULLY_RESOLVED_FILE_PATH} in last" ${SLEEP_TIME_SECONDS} "seconds"
    fi
    else
      echo "Config map ${FILE_PATH} doesn't exist. No action taken"
    fi
    sleep ${SLEEP_TIME_SECONDS}
  done
}

# Start periodic file check
Periodically_Read_Ronfigfile "/etc/global-config/log.persistenceConnectionPool" 10
```

It needs to be called from the entrypoint of the image and run permanently in the background.