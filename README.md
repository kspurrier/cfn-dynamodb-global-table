# cfn-dynamodb-global-table
---------------------------
This custom resource is meant to serve as a POC for the automation of DynamoDB Global Table creation.

Global Tables provide performant, fully managed, highly available, cross-region, multi-master DynamoDB Tables.  On top of all those benefits, we continue to get to leverage all the good parts of DynamoDB without having to change code that already uses the DynamoDB APIs.  

There are a few pre-requisites and things to take into consideration before creating a DynamoDB Global Table.
* You must have matching, empty DynamoDB Tables in each region where you want to configure replication.
* DynamoDB streams must be enabled (new and old images) on the tables.
* A global table cannot be edited if any of its region / replica tables have been populated.
* You need to know exactly where you want your data replicated up front.
* You cannot disable replication to a table and re-enable it without breaking said replication. It may look like it works, but replication to that region / replica will cease to function based on my most recent testing.

Adding a region / replica table or converting an existing DynamoDB table with data already in it will require backing up the data from the existing DynamoDB table, re-creating new empty tables in the desired region(s) and then re-playing / inserting the data so it can be replicated across the Global Table.  If this becomes a common occurence or desired pattern, it shouldn't be very hard to automate.  The currently available DynamoDB restore functionality (2018-09) only supports restoring into a new table in a single region, so you cannot restore into a preconfigured Global Table, and would have to build this functionality.

Testing
-------
Create a replica DynamoDB table in us-west-2 (the default replica region in the main CloudFormation template)
```
aws --region us-west-2 cloudformation deploy --template-file DynamoDBReplicaTable.yml --stack-name DynamoDBGlobalTableTest
```

Package the CloudFormation template that includes the DynamoDB Global Table custom resource and create the stack using the default values
```
aws --region us-east-1 cloudformation package --template-file DynamoDBGlobalTable.yml --s3-bucket dynamodb-global-test --output-template-file DynamoDBGlobalTable-packaged.yml
aws --region us-east-1 cloudformation deploy --template-file DynamoDBGlobalTable-packaged.yml --stack-name DynamoDBGlobalTableTest --capabilities CAPABILITY_NAMED_IAM
```

Add a test value to the GlobalTable via the region us-east-1
```
aws dynamodb put-item --table-name TestTable --item '{"ExKey": {"S":"us-east-1 test"}}' --region us-east-1
```

Scan the DynamoDB table to verify that the item exists in both regions
```
aws dynamodb scan --table-name TestTable --region us-west-2
aws dynamodb scan --table-name TestTable --region us-east-1
```

Add a test value to the GlobalTable via the replica region us-west-2
```
aws dynamodb put-item --table-name TestTable --item '{"ExKey": {"S":"us-west-2 test"}}' --region us-west-2
```

Scan the DynamoDB table to verify that both new items exist in each regions
```
aws dynamodb scan --table-name TestTable --region us-west-2
aws dynamodb scan --table-name TestTable --region us-east-1
```


Testing Cleanup
---------------
```
aws cloudformation delete-stack --stack-name DynamoDBGlobalTableTest --region us-west-2
aws cloudformation delete-stack --stack-name DynamoDBGlobalTableTest --region us-east-1
aws dynamodb delete-table --table-name TestTable --region us-west-2
aws dynamodb delete-table --table-name TestTable --region us-east-1
```
