Workshop Instructions:

Pre-requisites:
---
1. Make sure you have an AWS account
    1. Log in to aws.amazon.com
    2. Click on `Sign in to the Console` button on the top right
    3. Enter your Amazon username and password
    4. Click on `Sign In`

2. Make sure you have installed Python3 on your computer
    1. Go to [this link](https://www.python.org/downloads/)
    2. Download the latest version of Python3
    
3. Install npm. 
    1. Go to https://www.npmjs.com/get-npm
    2. Click on the `Download Node.js and npm` button
    3. Download the Latest Current version
---

Setup:
---

1. Open your terminal and navigate to the location that you want to create your project

```
cd ~
```

2. Create a directory and navigate into it
```
mkdir ghc2020_aws_workshop
cd ghc2020_aws_workshop
```

3. Download, install and configure the Amplify CLI:
```
npm install -g @aws-amplify/cli 
amplify configure
```

4. Create a new React project
```
npx create-react-app aws-community-news-bulletin
```

5. Navigate into the folder
```
cd aws-community-news-bulletin
```

6. Initialize AWS Amplify. 
```
amplify init
```
Answer the questions as following:
```
? Enter a name for the project ghc2020NewsBulletin
? Enter a name for the environment dev
? Choose your default editor: None
? Choose the type of app that you're building javascript
Please tell us about your project
? What javascript framework are you using react
? Source Directory Path:  src
? Distribution Directory Path: build
? Build Command:  npm run-script build
? Start Command: npm run-script start
```
Once this is done, call the following command on your CLI:
```
npm i aws-amplify
```
The `amplify init` command initializes the project, sets up deployment resources in the cloud, and makes your project ready for Amplify.

At this point your project set up is done. Let's dive into adding new components to our service!

---
### Set up storage for the service:
---

We first want to add a Dynamo DB table for our Community News Bulletin. Let's create a table called "Stories" which will store the different posts/news articles to be displayed on the website.

AWS Amplify allows us to easily create a Dynamo DB using CLI commands:

1. Let's start by creating a table 'Stories'
```
amplify add storage
```
Answer the questions as follows. Column names are case sensitive so please use the same case in your responses:
```
? Please select from one of the below mentioned services: NoSQL Database

Welcome to the NoSQL DynamoDB database wizard
This wizard asks you a series of questions to help determine how to set up your NoSQL database table.

? Please provide a friendly name for your resource that will be used to label this category in the project: Stories
? Please provide table name: Stories

You can now add columns to the table.

? What would you like to name this column: ID
? Please choose the data type: string
? Would you like to add another column? Yes
? What would you like to name this column: Title
? Please choose the data type: string
? Would you like to add another column? Yes
? What would you like to name this column: Content_url
? Please choose the data type: string
? Would you like to add another column? Yes
? What would you like to name this column: Num_likes
? Please choose the data type: number
? Would you like to add another column? No

Before you create the database, you must specify how items in your table are uniquely organized. You do this by specifying a primary key. The primary key uniquely identifies each item in the table so that no two items can have the same key. This can be an individual column, or a combination that includes a primary key and a sort key.

To learn more about primary keys, see:
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey

? Please choose partition key for the table: ID
? Do you want to add a sort key to your table? No

You can optionally add global secondary indexes for this table. These are useful when you run queries defined in a different column than the primary key.
To learn more about indexes, see:
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.SecondaryIndexes

? Do you want to add global secondary indexes to your table? No
? Do you want to add a Lambda Trigger for your Table? No
```
Your Dynamo DB storage is ready!

---

2. Let's add an S3 bucket to store the news articles for your website:
    1. Navigate to the [AWS Console](https://console.aws.amazon.com/console/home)
    2. In the search bar under `Find Services`, type `S3`
    3. Click on S3. You will navigate into the Amazon S3 Console.
    4. Click on `Create bucket`
    5. In the `General Configuration` section:
        1. Type in a bucket name like `<username>_ghc_2020_news_bulletin`
        2. Replace `<username>` with a unique name. All S3 bucket names have to be unique globally
        3. Choose the region that is closest to where you are based. For example, I live in Seattle so I chose `US West (Oregon) us-west-2`
    6. Don't make any other changes and click on `Create Bucket` at the bottom of the page
    
Your S3 bucket for storing new stories is ready!

---
## CreateStory API:
---
The CreateStory API will allow you to create a new article/story for your news bulletin. These will be stored in the two
storage resources we just created, which in turn will serve the readStories API that we will be creating next.
Instructions:

1. Navigate to your terminal and type the following command:
```
amplify add api
```
Answer the following questions as follows:
```
? Please select from one of the below mentioned services: REST
? Provide a friendly name for your resource to be used as a label for this category in the project: createStory
? Provide a path (e.g., /book/{isbn}): /story
? Choose a Lambda source Create a new Lambda function
? Provide a friendly name for your resource to be used as a label for this category in the project: createStory
? Provide the AWS Lambda function name: createStory
? Choose the runtime that you want to use: Python
Only one template found - using Hello World by default.
? Do you want to access other resources in this project from your Lambda function? Yes
? Select the category (Press <space> to select, <a> to toggle all, <i> to invert selection)
❯◯ storage
```

2. Navigate to your IDE. You should now see the backend code for the readStories API in the `<project_name>/amplify/backend/function` directory
3. Your lambda handler code is present in the `src` folder within the `createStory` directory
4. Let's navigate to the `index.py` file under `src` and replace the boilerplate code with the following:
```
import boto3
import boto3.dynamodb
from botocore.client import Config
import logging
import uuid
import json

TABLE_STORIES = "Stories-dev"
S3_BUCKET_NAME = "ghc2020-test1"

def handler(event, context):
  logging.info('Creating Dynamo DB client')
  print("Received event: {}".format(event))
  dynamodb_client = boto3.resource('dynamodb')
  stories_table = dynamodb_client.Table(TABLE_STORIES)

  s3_client = boto3.resource('s3', config=Config(signature_version='s3v4'))
  s3_bucket = s3_client.Bucket(S3_BUCKET_NAME)

  event_body = json.loads(event['body'])


  id = "story-"+str(uuid.uuid4())

  # store text content in s3
  content = event_body['content']
  content_key = event_body['title'].strip().replace(" ", "-")+".txt"
  s3_bucket.put_object(Key=content_key, Body=content, ContentType='text/plain')
  print('Stored content in s3')

  stories_table.put_item(
    Item={
      'id': id,
      'title': event_body['title'],
      's3Url': content_key,
      'number_of_likes': 0
    }
  )
  print('Stored record in Dynamo DB')
  return True
```
5. Now, you want to make sure that your AWS Lambda has the permissions to access your S3 bucket and Dynamo DB storage.
We can add permissions as follows:
    1. Navigate to the `createStory-cloudformation-template.json` file in the `createStory` directory
    2. Replace the `lambdaexecutionpolicy` section with the following:
 ```
"lambdaexecutionpolicy": {
      "DependsOn": [
        "LambdaExecutionRole"
      ],
      "Type": "AWS::IAM::Policy",
      "Properties": {
        "PolicyName": "lambda-execution-policy",
        "Roles": [
          {
            "Ref": "LambdaExecutionRole"
          }
        ],
        "PolicyDocument": {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "ReadWriteTable",
              "Effect": "Allow",
              "Action": [
                "dynamodb:BatchGetItem",
                "dynamodb:GetItem",
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:BatchWriteItem",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem"
              ],
              "Resource": "arn:aws:dynamodb:*:*:table/Stories-dev"
            },
            {
              "Sid": "s3Access",
              "Action": [
                "s3:*"
              ],
              "Effect": "Allow",
              "Resource": [
                "arn:aws:s3:::ghc2020-test1",
                "arn:aws:s3:::ghc2020-test1/*"
              ]
            },
            {
              "Effect": "Allow",
              "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
              ],
              "Resource": {
                "Fn::Sub": [
                  "arn:aws:logs:${region}:${account}:log-group:/aws/lambda/${lambda}:log-stream:*",
                  {
                    "region": {
                      "Ref": "AWS::Region"
                    },
                    "account": {
                      "Ref": "AWS::AccountId"
                    },
                    "lambda": {
                      "Ref": "LambdaFunction"
                    }
                  }
                ]
              }
            }
          ]
        }
      }
    }
```

6. Now you are ready to push your resources to the cloud. Go to your CLI and type:
```
amplify push
```
This command will prompt you for confirmation. Type Yes and push the created resources to your AWS account.
6. Wait until the command succeeds.
Great, your create story API is ready! We'll test this out on AWS Console soon!

---
## ReadStories API:
---
The ReadStories API will allow you to read all available stories/news posts that have been saved in the backend.
1. Navigate to your terminal and type the following command:
```
amplify add api
```
Answer the following questions as follows:
```
? Please select from one of the below mentioned services: REST
? Provide a friendly name for your resource to be used as a label for this category in the project: readStories
? Provide a path (e.g., /book/{isbn}): /readStories
? Choose a Lambda source Create a new Lambda function
? Provide a friendly name for your resource to be used as a label for this category in the project: readStories
? Provide the AWS Lambda function name: readStories
? Choose the runtime that you want to use: Python
Only one template found - using Hello World by default.
? Do you want to access other resources in this project from your Lambda function? Yes
? Select the category (Press <space> to select, <a> to toggle all, <i> to invert selection)
❯◯ storage
```

2. Navigate to your IDE. You should now see the backend code for the readStories API in the `<project_name>/amplify/backend/function` directory
3. Your lambda handler code is present in the `src` folder within the `readStories` directory
4. Let's navigate to the `index.py` file under `src` and replace the boilerplate code with the following:
```
import boto3
import boto3.dynamodb
import decimal
from botocore.client import Config
import logging
import json

TABLE_STORIES = "Stories-dev"
S3_BUCKET_NAME = "ghc2020-test1"

def handler(event, context):
  logging.info('Creating Dynamo DB client')
  print("Received event: {}".format(event))
  dynamodb_client = boto3.resource('dynamodb')
  stories_table = dynamodb_client.Table(TABLE_STORIES)

  s3_client = boto3.resource('s3', config=Config(signature_version='s3v4'))
  s3_bucket = s3_client.Bucket(S3_BUCKET_NAME)

  stories = stories_table.scan()

  for story in stories['Items']:
      print("story: {}".format(story))
      txt_content = s3_client.Object(S3_BUCKET_NAME, story['s3Url']).get()['Body'].read().decode('utf-8').splitlines()
      print("content: {}".format(txt_content))
      story['content'] = txt_content
      print("story with content: {}".format(story))

  body = {
    "stories": stories['Items']
  }

  response = {
    "statusCode": 200,
    "body": json.dumps(body, default=decimal_default),
    "headers": {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "POST, GET, OPTIONS, PUT, DELETE",
        "Access-Control-Allow-Headers": "Origin, Content-Type, Accept, Authorization, X-Request-With"
    }
  }

  return response

# Json does not decode decimal values by default.
# We can remove this method by storing num_likes as string values
def decimal_default(obj):
  if isinstance(obj, decimal.Decimal):
    return float(obj)
  raise TypeError
```
5. 5. Now, you want to make sure that your AWS Lambda has the permissions to access your S3 bucket and Dynamo DB storage.
   We can add permissions as follows:
       1. Navigate to the `readStories-cloudformation-template.json` file in the `readStories` directory
       2. Replace the `lambdaexecutionpolicy` section with the following:
    ```
   "lambdaexecutionpolicy": {
         "DependsOn": [
           "LambdaExecutionRole"
         ],
         "Type": "AWS::IAM::Policy",
         "Properties": {
           "PolicyName": "lambda-execution-policy",
           "Roles": [
             {
               "Ref": "LambdaExecutionRole"
             }
           ],
           "PolicyDocument": {
             "Version": "2012-10-17",
             "Statement": [
               {
                 "Sid": "ReadWriteTable",
                 "Effect": "Allow",
                 "Action": [
                   "dynamodb:BatchGetItem",
                   "dynamodb:GetItem",
                   "dynamodb:Query",
                   "dynamodb:Scan",
                   "dynamodb:BatchWriteItem",
                   "dynamodb:PutItem",
                   "dynamodb:UpdateItem"
                 ],
                 "Resource": "arn:aws:dynamodb:*:*:table/Stories-dev"
               },
               {
                 "Sid": "s3Access",
                 "Action": [
                   "s3:*"
                 ],
                 "Effect": "Allow",
                 "Resource": [
                   "arn:aws:s3:::ghc2020-test1",
                   "arn:aws:s3:::ghc2020-test1/*"
                 ]
               },
               {
                 "Effect": "Allow",
                 "Action": [
                   "logs:CreateLogGroup",
                   "logs:CreateLogStream",
                   "logs:PutLogEvents"
                 ],
                 "Resource": {
                   "Fn::Sub": [
                     "arn:aws:logs:${region}:${account}:log-group:/aws/lambda/${lambda}:log-stream:*",
                     {
                       "region": {
                         "Ref": "AWS::Region"
                       },
                       "account": {
                         "Ref": "AWS::AccountId"
                       },
                       "lambda": {
                         "Ref": "LambdaFunction"
                       }
                     }
                   ]
                 }
               }
             ]
           }
         }
       }
   ```

6. Go to your CLI and type
```aidl
amplify push
```
This command will prompt you for confirmation. Type Yes and push the created resources to your AWS account.
7. Wait until the command succeeds.

8. Once the amplify push command is successful, navigate to your AWS console and to API Gateway. Make sure you are in the same region that you chose to create your AWS resources. YOu can change your region by clicking on the region in the top right corner and clicking on a different region from the dropdown.
9. You should be able to see your `readStories` API here. Click on it.
10. Click on `ANY` under `/stories`
11. Click on the `TEST` button that shows up 