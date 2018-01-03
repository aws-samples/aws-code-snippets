### gd_setup_master.py

This script will setup GuardDuty on a master account, and invite a member account.  It will then wait for the invite to be accepted by the member account.

Usage: ```python3 gd_setup_master.py  --member_account_id 123456789012 --member_account_email MemberAccountRoot@example.com```

```python
# This script will setup GuardDuty on the master account and invite a member account.
# Contributed by Assaf Namer, namera@amazon.com
#
# Run this script using master account access-key and secret-access key
#
# Usage: python3 gd_setup_master.py  --member_account_id 123456789012 --member_account_email MemberAccountRoot@example.com

import boto3
import json
import sys
import argparse
import time

def main(arguments):

    parser = argparse.ArgumentParser(
        description=__doc__,
        formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('--member_account_id',required=True)
    parser.add_argument('--member_account_email',required=True)

    args = parser.parse_args(arguments)

    print ("Member account id: " , args.member_account_id )

    client = boto3.client('guardduty')

    #Find out if GuardDuty already enabled:
    detectors_list = client.list_detectors()

    if not detectors_list["DetectorIds"]:
        print ("GuardDuty is not enabled ... enabling GuardDuty on master account")
        response = client.create_detector(Enable=True);
        # Save DetectorID handler
        DetectorId = response["DetectorId"]
    else:
        print("GuardDuty already enabled on account")
        DetectorId = detectors_list['DetectorIds'][0]

    # Do error handling here

    # print all Detectorts
    detectors_list = client.list_detectors()
    print ("Detector lists: ")
    for x in detectors_list["DetectorIds"]:
        print(x, end=" ")


    # invite an account
    print ("\nInviting member account " + args.member_account_id )

    invite_member = client.create_members(
        AccountDetails=[
            {
                'AccountId': args.member_account_id,
                'Email': args.member_account_email
            },
        ],
        DetectorId=DetectorId
    )

    gd_members = client.get_members(
        AccountIds=[
            args.member_account_id,
        ],
        DetectorId=DetectorId
    )

    # the future member account is now staged 
    print ("Memeber account RelationshipStatus: " + gd_members['Members'][0]['RelationshipStatus'])

    # Invite members account(s)
    response = client.invite_members(
        AccountIds=[
            args.member_account_id,
        ],
        DetectorId=DetectorId,
        Message='Please join AWS GuardDuty master account'
    )

    gd_members = client.get_members(
        AccountIds=[
            args.member_account_id,
        ],
        DetectorId=DetectorId
    )

    # the future member account should be 'pending'
    print ("Memeber account RelationshipStatus: " + gd_members['Members'][0]['RelationshipStatus'])

    # wait in a loop for member account to accept the invitation
    invite_accepted = False

    while (invite_accepted == False):

    # At this point the member account needs to accept the invitation
    # if the invitation has been accepted then master accounnt RelationshipStatus should be "monitord"

        gd_members = client.get_members(
            AccountIds=[
                args.member_account_id,
                ],
                DetectorId=DetectorId
        )
        time.sleep (20);
        print ("Still waiting for member account to accept invite ...")
        # If invitation was accpted then member account should be 'monitored'
        print ("Memeber account RelationshipStatus: " + gd_members['Members'][0]['RelationshipStatus'])
        if 'Monitored' in gd_members['Members'][0]['RelationshipStatus']:
            invite_accepted = True

    print ("Invite has been accepted!");

if __name__ == '__main__':
    sys.exit(main(sys.argv[1:]))
```

### gd_print_findings.py

This script should be run in the master account.  It will read the findings and save them to a text file, as well as write them to a terminal.  Findings statistics are also displayed.  The output file name is optional.

Usage: ```python3 gd_print_findings.py --output_file_name my_file.txt```

```python
# This script prints GuardDuty findings from the master and member accounts. 
# An optional output file can be specified; the default file name is findings_output.txt.
# Contributed by Assaf Namer, namera@amazon.com
#
# Run this script using master account access-key and secret-access key
#
# Usage: python3 gd_print_findings.py --output_file_name my_file.txt
#
#
import boto3
import json
import sys
import argparse

def main(arguments):
    parser = argparse.ArgumentParser(
        description=__doc__,
        formatter_class=argparse.RawDescriptionHelpFormatter)
    
    parser.add_argument('--output_file_name', default='findings_output.txt' , required=False)

    args = parser.parse_args(arguments)

    client = boto3.client('guardduty')

    detectors_list = client.list_detectors()

    # print all Detectorts
    print ("Detector lists: ")
    for x in detectors_list["DetectorIds"]:
        print(x, end=" ")

    # get handler
    DetectorId = detectors_list['DetectorIds'][0]

    gd_findings = client.list_findings(
    DetectorId=DetectorId
    )

    # print all findings
    _print_finding = lambda DetectorId, FindingId: client.get_findings(
        DetectorId=DetectorId,
        FindingIds=[
            FindingId,
        ]
    )

    #keep findinds in a buffer - in case we'd like to do text manipulation later on
    findings_buffer=''
    # Print all findings in JSON format
    for _find in gd_findings['FindingIds']:
        _find = (_print_finding(DetectorId , _find ))
        findings_buffer +=  json.dumps(_find)

    
    # print results to a text file
    with open(args.output_file_name, 'w') as outfile:
        json.dump(findings_buffer, outfile)

    # print to terminal (comment out this line if the list is too long, use text file instead)
    print (findings_buffer)

   
    # print the count of findings for the given severity.
    findings_stat = client.get_findings_statistics(
        DetectorId=DetectorId,
        FindingStatisticTypes=[
            'COUNT_BY_SEVERITY',
        ]
    )

    print ('\n\nFindings Statistics: ' + json.dumps(findings_stat['FindingStatistics']))

if __name__ == '__main__':
    sys.exit(main(sys.argv[1:]))
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>