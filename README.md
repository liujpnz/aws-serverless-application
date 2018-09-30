# aws-serverless-application
An aws serverless application using s3, dynamodb, lambda, sns, sqs, cloudformation and cloudwatch compoments

- overview
  - architecture
  - tools
  - configuration

- environment
  - python
  - awscli
- setup
- s3
- dynamodb
- sns sqs
- labmda

# overview
##architecuture

## tools
  ### ubuntu
  ```
  jason@Server:~/serverless$ uname -a
  Linux Server 4.15.0-34-generic #37-Ubuntu SMP Mon Aug 27 15:21:48 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
  ```
  
  ### python
  ```
  python --version
  Python 3.6.6
  ```
  
  ### aws cli
  install aws cli via atp-get tools
  ```
  sudo apt-get install awscli
  
  ```
  ### boto3
  
  install the latest Boto 3 release via pip
  ```
  sudo pip install boto3
  ```
## configuration
  ### Credential
  Get Credential for your AWS account, which include Access key ID and Secret access key
  
  ### aws cli configure
  Before using Boto 3, you should set up authentication credential in aws cli, input your own credential.
  ```
  jason@Server:~/serverless$ aws configure
  AWS Access Key ID [****************IVPA]: 
  AWS Secret Access Key [****************vvoz]: 
  Default region name [ap-southeast-2]: 
  Default output format [None]: 
  ```
  
  After configuration, you can see configuration files in ~/.aws/
  
  ```
  jason@Server:~/.aws$ pwd
  /home/jason/.aws
  jason@Server:~/.aws$ cat config 
  [default]
  region = ap-southeast-2
  jason@Server:~/.aws$ cat credentials 
  [default]
  aws_access_key_id = ****************IVPA
  aws_secret_access_key = ****************vvoz
  ```
  ### test configuration
  You can use aws command to verify. The following command will print current user in you aws account.
  
  ```
  jason@Server:~/.aws$ aws iam list-users
{
    "Users": [
        {
            "Path": "/",
            "UserName": "api-user",
            "UserId": "AIDAJGWZFNWPZBMSS24HQ",
            "Arn": "arn:aws:iam::661332127240:user/api-user",
            "CreateDate": "2018-09-22T08:07:38Z"
        },
  ...
  
  ```

# Create S3 buckets
Create input and output buckets with specified regions, here we are using ap-southeast-2 as default regions.
Specify the bucket name, enter to use default value. 
*Notes: bucket name should be global unique*

```
jason@Server:~/serverless$ python 1create-s3-bucket.py
input bucket name: [jasons-python-input]
jasons-python-input
output bucket name: [jasons-python-output]
jasons-python-output
jasons-python-input bucket created.
jasons-python-output bucket created.

  ```



# Create DynamoDB tables
Create three dynamodb tables with specified key.
-Customers
-Transactions
-TotalAmount  
*For transactions table, we should enable stream with new and old images for lambda funcations.*

```
jason@Server:~/serverless$ python 2create-dynamodb-table.py 
dynamodb.Table(name='Customers')  status: CREATING
dynamodb.Table(name='Transactions')  status: CREATING
dynamodb.Table(name='TotalAmount')  status: CREATING
```

Three new tables are created.
```
jason@Server:~/serverless$ aws dynamodb list-tables
{
    "TableNames": [
        "Customers",
        "TotalAmount",
        "Transactions",
        "posts"
    ]
}
```

# Create SNS topic, SQS service

## SNS topic 
Create SNS topic TransactionAlert, record the TopicArn for further subscriptions.

```
jason@Server:~/serverless$ aws sns  create-topic --name TransactionAlert
{
    "TopicArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert"
}
```

## Subscribe an email address to SNS topic:

Subscribe an email address to TransactionAlert, you need to change to your own topic-arc and email address.

```
jason@Server:~/serverless$ aws sns subscribe --topic-arn arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert --protocol email --notification-endpoint jason.liu.dba@gmail.com
{
    "SubscriptionArn": "pending confirmation"
}
```
Check your email to confirm the subscription. Then the subscription succeed. 

```
jason@Server:~/serverless$ aws sns  list-subscriptions-by-topic --topic-arn arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert
{
    "Subscriptions": [
        {
            "SubscriptionArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert:f0e4e216-a148-4f73-ba09-e2c95c047912",
            "Owner": "661332127240",
            "Protocol": "email",
            "Endpoint": "jason.liu.dba@gmail.com",
            "TopicArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert"
        }
    ]
}
```



##create SQS service, and subscribe to SNS topic

Create SQS queue TransactionAlertSQS
```
jason@Server:~/serverless$ aws sqs create-queue --queue-name TransactionAlertSQS
{
    "QueueUrl": "https://ap-southeast-2.queue.amazonaws.com/661332127240/TransactionAlertSQS"
}
```

Get TransactionAlertSQS QueueArn attribute 
```
jason@Server:~/serverless$ aws sqs get-queue-attributes  --queue-url https://ap-southeast-2.queue.amazonaws.com/661332127240/TransactionAlertSQS --attribute-names QueueArn
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:ap-southeast-2:661332127240:TransactionAlertSQS"
    }
}
```

add subscription to SNS topic
--topic-arn is the arn of your SNS topic
--notification-endpoint is the arn of your SQS service

```
jason@Server:~/serverless$ aws sns subscribe --topic-arn arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert --protocol sqs --notification-endpoint arn:aws:sqs:ap-southeast-2:661332127240:TransactionAlertSQS
{
    "SubscriptionArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert:51ba3e3b-812b-4177-9f86-0fddc68ecbd6"
}
```


### list all subscriptions

from here, we can ..
```
jason@Server:~/serverless$ aws sns  list-subscriptions-by-topic --topic-arn arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert
{
    "Subscriptions": [
        {
            "SubscriptionArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert:f0e4e216-a148-4f73-ba09-e2c95c047912",
            "Owner": "661332127240",
            "Protocol": "email",
            "Endpoint": "jason.liu.dba@gmail.com",
            "TopicArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert"
        },
        {
            "SubscriptionArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert:51ba3e3b-812b-4177-9f86-0fddc68ecbd6",
            "Owner": "661332127240",
            "Protocol": "sqs",
            "Endpoint": "arn:aws:sqs:ap-southeast-2:661332127240:TransactionAlertSQS",
            "TopicArn": "arn:aws:sns:ap-southeast-2:661332127240:TransactionAlert"
        }
    ]
}
```


#setup
