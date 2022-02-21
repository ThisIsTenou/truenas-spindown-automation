#!/bin/bash

# ZFS variables
guid="00000000000000000000" # GUID of the pool this script should import/export

# Replication variables
replname="replicationtask-1234" # Name of the replication task this script should run

# PDU Variables
webhookurl="https://webhook.endpoint.local/enclosure-control" # Custom HTTP Endpoint to control disk enclosure
pdu1ip="192.168.0.1"
pdu1user="user1"
pdu1pwd="password1"
pdu1sct="1-4"
pdu2ip="192.168.0.2"
pdu2user="user2"
pdu2pwd="password2"
pdu2sct="1-4"

validate_job () {
  echo "Waiting for job to finish..."
  jobstate=$(midclt call core.get_jobs "[[\"id\",\"=\",$jobid]]" | jq -r ".[].state")
  until [[ "$jobstate" == "SUCCESS" ]] || [[ "$jobstate" == "FAILED" ]]; do
    case "$jobstate" in
      SUCCESS|FAILED|RUNNING)
        ;;
      *)
        echo "Jobstate cannot be parsed: $jobstate"
        return 1
        ;;
    esac
    sleep 5
    jobstate=$(midclt call core.get_jobs "[[\"id\",\"=\",$jobid]]" | jq -r ".[].state")
  done
  unset jobid
  case "$jobstate" in
    SUCCESS)
    echo "Task finished successfully."
    return 0
    ;;
    FAILED)
    echo "Task failed."
    return 1
    ;;
    *)
    echo "Jobstate cannot be parsed: $jobstate"
    return 1
    ;;
  esac
}

import_pool () {
  echo "Starting import for pool with guid $guid"
  # Check if pool already imported
  importpoolid=$(midclt call pool.query | jq ".[] | select(.guid == \"$guid\").id")
  if [[ -n "$importpoolid" ]]; then
    echo "Pool is already imported (ID $importpoolid), skipping..."
  else
    # Import pool with specified guid
    jobid=$(midclt call pool.import_pool "{\"guid\":\"$guid\"}")
    echo "Current job id: $jobid"
    # Wait until job completed
    validate_job
  fi
}

export_pool () {
  echo "Export for pool with guid \"$guid\" started."
  # Find pool to export and check if it's not empty
  exportpoolid=$(midclt call pool.query | jq ".[] | select(.guid == \"$guid\").id")
  if [[ -z "$exportpoolid" ]]; then
    echo "Pool is not imported, skipping export."
  else
    echo "Proceeding with pool id $exportpoolid"
    # Export pool with specified id
    jobid=$(midclt call pool.export $exportpoolid '{"cascade":false,"destroy":false}')
    echo "Current job id: $jobid"
    # Wait until export completed
    validate_job
  fi
}

start_replication () {
        echo "Searching for replication task..."
        # Find replication task id by name
        replid=$(midclt call replication.query | jq ".[] | select(.name == \"$replname\").id")
        [[ -z "$replid" ]] && { echo "Replication task id could not be determined"; return 1; }
        echo "Proceeding with replication task id $replid"
        # Start replication task
        jobid=$(midclt call replication.run $replid)
        echo "Current job id: $jobid"
        # Wait until replication completed
        validate_job
}

start_disks () {
  echo "Calling webhook to start disks."
  curl -s -k --data "{\"host\":\"$pdu1ip\",\"user\":\"$pdu1user\",\"password\":\"$pdu1pwd\",\"action\":\"on\",\"sockets\":\"$pdu1sct\"}" --output "/dev/null" "$webhookurl"
  curl -s -k --data "{\"host\":\"$pdu2ip\",\"user\":\"$pdu2user\",\"password\":\"$pdu2pwd\",\"action\":\"on\",\"sockets\":\"$pdu2sct\"}" --output "/dev/null" "$webhookurl"
  echo "Disks spinning up, sleeping for 2 minutes..."
  sleep 120
}

stop_disks () {
  curl -s -k --data "{\"host\":\"$pdu1ip\",\"user\":\"$pdu1user\",\"password\":\"$pdu1pwd\",\"action\":\"off\",\"sockets\":\"$pdu1sct\"}" --output "/dev/null" "$webhookurl"
  curl -s -k --data "{\"host\":\"$pdu2ip\",\"user\":\"$pdu2user\",\"password\":\"$pdu2pwd\",\"action\":\"off\",\"sockets\":\"$pdu2sct\"}" --output "/dev/null" "$webhookurl"
}

case "$1" in
  start_disks)
    start_disks
        ;;
  import_pool)
    import_pool
        ;;
  start_replication)
    start_replication
        ;;
  export_pool)
    export_pool
        ;;
  stop_disks)
    stop_disks
        ;;
  backuptask)
    start_disks
    import_pool
    start_replication
    export_pool
    stop_disks
        ;;    
  start_disks_and_import)
    start_disks
        import_pool
        ;;
  export_and_stop_disks)
    export_pool
    stop_disks
        ;;    
  *)
    echo "Please provide one of the following arguments to run this script:"
        echo "start_disks, import_pool, start_replication, export_pool, stop_disks, backuptask, start_disks_and_import, stop_disks_and_export"
        ;;
esac
