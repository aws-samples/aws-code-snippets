Get all messages from an SQS queue (short polling):

```sh
QUEUE=$SQS_QUEUE 			# Provided by CodeBuild environment
URL=`aws sqs get-queue-url --queue-name=$QUEUE --output text`
MSGFILE=/tmp/msgs-$$

function receive_msgs() {
	i=1
	echo Calling aws sqs receive-message \#$i...
	aws sqs receive-message --queue-url $URL --attribute-names All \
	  --message-attribute-names All --max-number-of-messages 10 > $MSGFILE-$i.json
	if [ $? -ne 0 ]
	then
		echo "aws sqs receive-message command failed; exiting."
		exit 1
	fi
	[ `cat $MSGFILE-$i.json|wc -l` -eq 0 ] && echo "No messages." && exit 0 # Exit if there were no messages
	while [ `cat $MSGFILE-$i.json|wc -l` -gt 0 ] # If >0 lines, at least 1 message was received
	do
		i=`expr $i + 1`
		echo Calling aws sqs receive-message \#$i...
		aws sqs receive-message --queue-url $URL --attribute-names All \
		  --message-attribute-names All --max-number-of-messages 10 > $MSGFILE-$i.json
		if [ $? -ne 0 ]
		then
			echo "aws sqs receive-message command failed; exiting."
			exit 1
		fi
	done
	rm -f $MSGFILE-$i.json # the last file will have 0 lines (no messages) so remove it
}
```

Create a JSON to remove all received messages from the SQS queue:

```sh
MSGFILE=/tmp/msgs-$$
REMOVEFILE=/tmp/remove-msgs-$$

function create_msg_removal_json() {
	i=1
	for msgfiles in $MSGFILE-*
	do
		echo Creating message removal JSON \#$i...
		if [ `cat $msgfiles|wc -l` -gt 0 ] # Make sure there is a message in the file first
		then
			cat $msgfiles|jq --slurp '[.[] | .Messages[] | {Id:.MessageId, ReceiptHandle:.ReceiptHandle}]' > $REMOVEFILE-$i.json
			if [ $? -ne 0 ]
			then
				echo "Error creating batch message removal JSON; exiting."
				exit 2
			fi
			i=`expr $i + 1`
		fi
	done
}
```

Remove all received messages from the SQS queue:

```sh
QUEUE=$SQS_QUEUE 			# Provided by CodeBuild environment
URL=`aws sqs get-queue-url --queue-name=$QUEUE --output text`
REMOVEFILE=/tmp/remove-msgs-$$
REMOVERESULT=/tmp/result-$$

function remove_msgs() {
	i=1
	if ls $REMOVEFILE-* 1> /dev/null 2>&1; then # Make sure these files exist before executing.
		for removefiles in $REMOVEFILE-*
		do
			echo Calling aws sqs delete-message-batch \#$i...
			aws sqs delete-message-batch --queue-url $URL --entries file://$removefiles > $REMOVERESULT-$i.json
			if [ $? -ne 0 ]
			then
				echo "Error executing batch message removal; exiting."
				exit 3
			fi
			i=`expr $i + 1`
		done
	fi
}
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>