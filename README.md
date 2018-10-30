# Using Lambda Function with Vault
This document gives an example of integrating AWS Lambda Functions with Vault.  The Lambda function reads a database username and password from Vault and then inserts a row into an AWS RDS MySQL database called ExampleDB.

We use IAM roles so that when the Lambda function runs under the "lambda-vault-dev" role, it can only get the username and password from the "secret/dev/lambda/mysql/ExampleDB" path, and when it runs under the "lambda-vault-qa" role, it can only get the username and password from the "secret/qa/lambda/mysql/ExampleDB" path.

This example is based on the AWS [Tutorial: Configuring a Lambda Function to Access Amazon RDS in an Amazon VPC](https://docs.aws.amazon.com/lambda/latest/dg/vpc-rds.html), but I modified it to consume Lambda events, use Lambda environment variables, and authenticate against Vault and read the database username and password from it.

This example has a Lambda function written in Python that creates an Employee table and inserts rows into it. Another modification I made was to use "if not exists" when creating the database table so that using the function after the first time would not fail.

## Setting up the MySQL database in AWS RDS
The steps to do this are from [Creating a MySQL DB Instance and Connecting to a Database on a MySQL DB Instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.MySQL.html)

Please create an AWS RDS MySQL database in one of your VPCs with these properties:

Public Assessibility: yes
instance class: db.t2.small
instance identifier: "lambdavault"
master username: "vault"
password: "vault123"
Database name: ExampleDB
port: 3306

I tested with mysql 5.6.39 and 5.7.21.

After creating the RDS instance, record the endpoint.  For instance, mine was `lambdavault.cihgglcplvpp.us-east-1.rds.amazonaws.com`

If you are unable to create your RDS instance with istance identifier equal to "lambdavault" (because I have already used it), pick something similar instead.

You should then be able to connect to the database with a command like:
`mysql -h lambdavault.cihgglcplvpp.us-east-1.rds.amazonaws.com -P 3306 -u vault -p`
Then provide your password, "vault123". Use "exit" to quit mysql.

For later testing purposes, add two users, "dev" and "qa":

```
use ExampleDB
CREATE USER 'dev'@'%' IDENTIFIED BY 'vault123dev';
GRANT ALL PRIVILEGES ON `%`.* TO dev@`%`;
CREATE USER 'qa'@'%' IDENTIFIED BY 'vault123qa';
GRANT ALL PRIVILEGES ON `%`.* TO qa@`%`;
```

## Configure Vault
You need to do some configuration in Vault to support the AWS authentication by Lambda functions executing under the dev and qa roles.

We'll use two Vault policies, dev-policy.hcl and qa-policy.hcl which are attached. While they have some extra stanzas to let dev and qa users use the Vault UI to interact with their secrets, the most important things in them are the following:

dev-policy.hcl:
```
# Access to secret/dev
path "secret/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

qa-policy.hcl:
```
# Access to secret/qa
path "secret/qa/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

Let's import the Vault policies, dev-policy.hcl and qa-policy.hcl, creating dev and qa policies respectively by running these commands:

```
vault policy write dev dev-policy.hcl
vault policy write qa qa-policy.hcl
```

You need to store the dev and qa usernames and associated passwords in Vault. Please issue the following commands or use the Vault UI to enter the secrets:

```
vault write secret/dev/lambda/mysql/ExampleDB db_user=dev db_pwd=vault123dev
vault write secret/qa/lambda/mysql/ExampleDB db_user=qa db_pwd=vault123qa
```

You also need to configure the AWS authentication method on Vault with these commands. I've commented out the command that stores AWS keys on the auth/aws/config/client path under the assumption that the Vault server is running in EC2 and can use the EC2 instance's own role to support the AWS iam authentication method.  If you are running your Vault server on an EC2 instance and do not provide AWS keys, then assign that instance an IAM role that has the vault-aws-authentication IAM policy shown below.

```
aws auth enable aws

#vault write auth/aws/config/client access_key=<access_key> secret_key=<secret_key>

vault write auth/aws/role/lambda-dev-role auth_type=iam bound_iam_principal_arn=arn:aws:iam::<account_id>:role/lambda-vault-dev  policies=dev max_ttl=24h

vault write auth/aws/role/lambda-qa-role auth_type=iam bound_iam_principal_arn=arn:aws:iam::<account_id>:role/lambda-vault-qa  policies=qa max_ttl=24h
```

## Create AWS Lambda Function
You can now create your Lambda Function. I did this by first creating a Lambda function deployment package following  [Step 2.1](https://docs.aws.amazon.com/lambda/latest/dg/vpc-rds-deployment-pkg.html), then creating an IAM Execution Role for the Lambda function following [Step 2.2](https://docs.aws.amazon.com/lambda/latest/dg/vpc-rds-create-iam-role.html), and then creating the Lambda function itself following [Step 2.3](https://docs.aws.amazon.com/lambda/latest/dg/vpc-rds-upload-deployment-pkg.html).

But you can skip most of that since I've provided you with the code for the Lambda function.

### Create IAM Roles
We actually want 2 IAM roles, one for dev and one for QA. So, do create IAM roles (using the AWS Console) lambda-vault-dev and lambda-vault-qa with policies AWSLambdaVPCAccessExecutionRole and vault-aws-authentication where the latter should have these permissions:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "iam:GetInstanceProfile",
                "iam:GetUser",
                "iam:GetRole"
            ],
            "Resource": "*"
        }
    ]
}
```

The first policy is a pre-existing one.  You should create the second one from https://console.aws.amazon.com/iam. Select Policies on the left-side menu, click the "Create policy" button, click the JSON tab, and past the above permissions into the JSON editor, replacing any text already there. Click the "Review policy" button. Name the policy "vault-aws-authentication", provide a description, and click the "Create policy" button.

To create the roles in the AWS Console, select Roles" from the left-side menu and click the "Create role" button. Select "AWS service" from the top of the screen and then "Lambda". Then click the "Next: Permissions" button. Type "Lambda" into the policy filter and check the box for the AWSLambdaVPCAccessExecutionRole role. Then change the filter to "vault" and check the box for the vault-aws-authentication role. Click the "Next: Review" button. Name your role "lambda-vault-dev" and give it a description like "Allows Lambda functions to operate in the development environment". Finally, click the "Create role" button.

Repeat this process for the lambda-vault-qa role.

Note that ARNs of your roles. They will be something like "arn:aws:iam::362391645459:role/lambda-vault-dev" and "arn:aws:iam::362391645459:role/lambda-vault-qa".

### Create the Lambda Function
You can now create a Lambda Function in the AWS Console. Navigate to https://console.aws.amazon.com/lambda, click the "Create a function" button, and then select the "Author from scratch" option.

Name your function "InsertEmployees", specify "Python 2.7" as the Runtime, and specify "Create a custom role" for the Role.  Specify "Chose an existing role" and select the "lambda-vault-dev" role. Then click the "Create function" button.

Scroll down to the "Function code" section, set the Handler to "app.handler", and click on the "Code entry type" drop-down (that defaults to "Edit code inline"). Select "Upload a .ZIP file", click the Upload button, navigate to the app.zip file from this project, select it, and click the Choose button. Then click the orange Save button at the top of the screen.

You now have a Lambda function that can talk to Vault and your RDS MySQL database.  But we need to set some environment variables for it.

Set "ENV" to "dev".
Set "RDS_HOST" to the endpoint of your RDS MySQL database.
Set "VAULT_ADDR" to the address of your Vault server including the port so that it is something like "http://<dns>:8200" or "http://<IP>:8200".

Save your function again.

## Testing your Lambda Function
Before testing your Lambda function, create a test event in the AWS Lambda Management console. In the screen showing the InsertEmployees Lambda function, click the "Select a test event.." field to the left of the Test button and select the "Configure test events" option. You can use the "Hello World" template, name the event AnEmployee, and then edit the event to just have row item, remove the comma from that item, change the key to "name" and the value to any single name you want such as your first name. Click the Create button at the bottom of the screen.

You can now test your function as long as the "AnEmployee" test event is selected in the drop-down next to the Test button.

Simply click the Test button to test your function.

If it is successful, you will see a green box which you can open up and see the output from the function along with some limited logging. You can also click on a Logs link to see more logging for each execution of your Lambda function in CloudWatch.

Validate that your function works with the environment variable ENV set to "dev" when running under the lambda-vault-dev role. If you change ENV to "qa", save, and test again, it should fail.

Switch the ENV variable to "qa" and change the role to "lambda-vault-qa".  Save the function.  Now, when you test, it should work again.

## The Lambda Function Code
For reference, we've included the Lambda function code here. The comments and code should make it pretty clear what the function is doing.

```
import sys
import logging
import pymysql
import hvac
import json
import os
import boto3

from base64 import b64decode

logger = logging.getLogger()
logger.setLevel(logging.INFO)

#rds settings
rds_host  = os.environ['RDS_HOST']

# Get database user name, and password from Vault
try:
    client = hvac.Client(url=os.environ['VAULT_ADDR'])
    session = boto3.Session()
    credentials = session.get_credentials()
    client.auth_aws("lambda-" + os.environ['ENV'] + "-role", credentials.access_key, credentials.secret_key, credentials.token)
    db_values = client.read('secret/' + os.environ['ENV'] + '/lambda/mysql/ExampleDB')
    db_name = 'ExampleDB'
    name = db_values['data']['db_user']
    logger.info("DB user is: " + str(name))
    password = db_values['data']['db_pwd']
    logger.info("DB password is: " + str(password))
except:
    logger.error("ERROR: Unexpected error: Could not connect to Vault.")
    sys.exit()

try:
    conn = pymysql.connect(rds_host, user=name, passwd=password, db=db_name, connect_timeout=5)
except:
    logger.error("ERROR: Unexpected error: Could not connect to MySql instance.")
    sys.exit()

logger.info("SUCCESS: Connection to RDS mysql instance succeeded")
def handler(event, context):
    """
    This function fetches content from mysql RDS instance
    """
    with conn.cursor() as cur:
        cur.execute("create table if not exists Employee ( EmpID  int NOT NULL AUTO_INCREMENT, Name varchar(255) NOT NULL, PRIMARY KEY (EmpID))")
        cur.execute('select count(*) from Employee')
        num_rows = cur.fetchone()
        logger.info("Employee table had " + str(num_rows) + " rows")
        cur.execute('insert into Employee (Name) values(%s)', (event['name']))
        conn.commit()
        cur.execute('select * from Employee where EmpID > %s', (num_rows) )
        item_count = 0
        for row in cur:
            item_count += 1
            logger.info(row)

    return "User %s with password %s added %d item(s) to the Employee table" %(name, password, item_count)

```
