Fetch the S3 prefix list IDs for S3 in all AWS regions. These can then be used in route tables and security groups to allow access to S3:

```sh
#!/bin/bash

# Fetch the S3 prefix list IDs for S3 in all AWS regions.
# These can then be used in route tables and security groups to allow access to S3

aws ec2 describe-regions --query 'Regions[*].RegionName' | \
jq -r '.[]' | \
while read REGION
do
  echo ==== $REGION ====
  aws ec2 describe-prefix-lists \
    --region $REGION \
    --filters Name=prefix-list-name,Values=com.amazonaws.$REGION.s3 \
    --query 'PrefixLists[0].PrefixListId'
done
```

### Snapshot Cleanup
This script is designed to lookup and delete snapshots by tag key/value pair of a certain age.

Required arguments:
* -r  [ AWS Region(s) ] ( Can specify multiple )
* -a  [ Account ID(s) to Query ] ( Can specify multiple )
* -k  [ AWS Tag:Key to lookup snapshots by ]
* -v  [ AWS Tag:Value to lookup snapshots by ]
* -d  [ Retention time in Days to delete snapshots ]

Optional Arguments:
* -p  [ AWS Profile to use ]

Example Usage:
The following will delete all snapshots created by the tag CreatedBy:AutomatedBackups older than 7 days in both us-west-2 and us-east-1

```sh
./snapshot-cleanup.sh -r "us-west-2 us-east-1" -k CreatedBy -v AutomatedBackups -a 201035249631 -d 7
```

```sh
#!/bin/bash
#---------------------------------------------------#
# Author: Chris Stobie                              #
# Contact: cjstobie@amazon.com                      #
#---------------------------------------------------#

spacer() {
    printf '%*s\n' "${COLUMNS:-$(tput cols)}" '' | tr ' ' -
}
help_func() {
    spacer
    printf "%s\n" "$0 Usage"
    spacer
    printf "%s\n" "REQUIRED ARGUMENTS"
    spacer
    printf "%s\n" " -r  [ AWS Region(s) ]"
    printf "%s\n" " -k  [ AWS Tag:Key to lookup snapshots by ]"
    printf "%s\n" " -v  [ AWS Tag:Value to lookup snapshots by ]"
    printf "%s\n" " -d  [ Retention time in Days to delete snapshots ]"
    printf "%s\n" " -a  [ Account ID(s) to Query ]"
    spacer
    printf "%s\n" "OPTIONAL ARGUMENTS"
    spacer
    printf "%s\n" " -p  [ AWS Profile to use ]"
    spacer
}
get_opts() {
    while getopts "hr:k:v:d:a:p:" opt; do
        case $opt in
            h) help_func; exit 0;;
            a) account_ids+=("$OPTARG") ;;
            r) regions+=("$OPTARG") ;;
            k) tag_key+=$OPTARG ;;
            v) tag_value=$OPTARG ;;
            d) retention_days=$OPTARG ;;
            p) aws_profile=$OPTARG ;;
        esac
    done
}
validate_input() {
    for var in regions tag_key tag_value account_ids retention_days; do
        if [[ -z ${!var} ]]; then
            spacer
            printf "%s\n" "Missing $var"
            help_func
            exit 1
        fi
    done
}
delete_snap() {
    printf "%s\n" "Deleting Snapshot [$id]"
    aws_com ec2 delete-snapshot --region $region --snapshot-id $id
}
aws_com() {
    if [[ -n $aws_profile ]]; then
        aws "$@" --profile $aws_profile
    else
        aws "$@"
    fi
}
cleanup_snapshots() {
    # Calculate retention time in Epoch Seconds
    for account in ${account_ids[@]}; do
        printf "%s\n" "Cleaning up snapshots in $account"
        for region in ${regions[@]}; do
            printf "%s\n" "Cleanig up snapshots in $region"
            retention_secs=$(date +%s --date "$retention_days days ago")
            while read -r time id; do

                time_secs=$(date "--date=$time" +%s)

                if (( $time_secs <= $retention_secs )); then
                    cnt=0
                    # Run this in a loop in case we hit CLI Thresholds
                    until delete_snap; do
                        # Retry the deletion 5 times before failing
                        if (( cnt == 5 )); then
                            printf "%s\n" "Max retry reached"
                            break
                        fi
                        # Backup in case we hit the CLI thresholds
                        printf "%s\n" "CLI Threshold Hit, Retrying..."
                        sleep 5
                        ((cnt++))
                    done
                fi

            done < <(aws_com ec2 describe-snapshots --owner-ids $account_id --region $region --output=text --filters "Name=tag:$tag_key,Values=$tag_value" --query 'Snapshots[*].[StartTime,SnapshotId]')
        done
    done
}
main() {
    get_opts "$@"
    validate_input
    cleanup_snapshots
}

main "$@"
```

### Get Instance ID's 
This script will get instance ID's by key/value pair

This script requires the following variables:
* REQUIRED | -v | tag-value     | Value of the tag you want to query against
* REQUIRED | -k | tag-key       | Key of the tag you want to query against
* OPTIONAL | -r | region        | AWS region to query against
* OPTIONAL | -o | output        | yml|yaml|bash|shell
* OPTIONAL | -a | ansible hosts | yml|yaml|bash|shell

Output Options:
  Default     | Line separated list of instance ID's
  Yaml        |  - instanceid
  Bash/Shell  |  instanceid instanceid instanceid
  
Example Usage:
  ./get-instance-ids -r us-west-2 -k envtype -v stg-db
  ./get-instance-ids -v stg-db

```sh
#!/bin/bash
#---------------------------------------------------------------------------#
# Author: Christopher Stobie                                                #
# Contact: cjstobie@amazon.com                                              #
#---------------------------------------------------------------------------#

spacer() {
    printf '%*s\n' "${COLUMNS:-$(tput cols)}" '' | tr ' ' -
}

_get-ids-help-func(){
    printf "%s\n" "This script requires the following variables:"
    spacer
    printf "%s\n" "REQUIRED | -v | tag-value     | Value of the tag you want to query against"
    printf "%s\n" "REQUIRED | -k | tag-key       | Key of the tag you want to query against"
    spacer
    printf "%s\n" "OPTIONAL | -r | region        | AWS region to query against"
    printf "%s\n" "OPTIONAL | -o | output        | yml|yaml|bash|shell"
    printf "%s\n" "OPTIONAL | -a | ansible hosts | yml|yaml|bash|shell"
    spacer
    printf "%s\n" "Output Options:"
    printf "%s\n" "  Default     | Line separated list of instance ID's"
    printf "%s\n" "  Yaml        |  - instanceid"
    printf "%s\n" "  Bash/Shell  |  instanceid instanceid instanceid" 
    spacer
    printf "%s\n" "Example Usage:"
    printf "%s\n" "  $0 -r us-west-2 -k envtype -v stg-db"
    printf "%s\n" "  $0 -v stg-db"
    spacer
    printf "%s\n" "Exiting..."
}
_get-ids-global-vars() {
    region=""
    key=""
    value=""
    output=""
}
_get-ids-opt-func() {
    local OPTIND
    while getopts "r:k:v:o:ah" opt; do
        case $opt in
            r) region=$OPTARG ;;
            k) key=$OPTARG ;;
            v) value=$OPTARG ;;
            o) output=$OPTARG ;;
            h) _get-ids-help-func; exit 0;;
        esac
    done
}
_get-ids-check-vars() {
    if [[ -z $region ]]; then
        region="us-west-2"
    fi
    if [[ -z $output  ]]; then
        output="txt"
    fi
    if [[ -z $value || -z $key ]]; then
        _get-ids-help-func
        exit 1
    fi
}
_get-ids-format() {
    if [[ $output =~ txt ]]; then
        printf "%s\n" "$instances"
    elif [[ $output =~ yml|yaml ]]; then
        printf "%s\n" "$instances"|sed 's/^/- /g'
    elif [[ $output =~ shell|bash ]]; then
        printf "%s\n" "$instances" | paste -sd" "
    fi
}
_get-ids-main() {
    instances=$(aws ec2 describe-instances --region $region --filters "Name=tag:$key,Values=$value" --query "Reservations[*].Instances[*].InstanceId" --output text)
    _get-ids-format
}
get-ids() {
    _get-ids-global-vars
    _get-ids-opt-func "$@"
    _get-ids-check-vars
    _get-ids-main
}

get-ids "$@"
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>

