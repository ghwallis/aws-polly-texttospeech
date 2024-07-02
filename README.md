## Serverless Text-to-Speech App with AWS Polly

### Overview

This project demonstrates how to build a serverless application using various AWS services to convert text to speech and store the resulting audio files. The application provides two methods – one for sending information about a new post, which is converted into an MP3 file, and one for retrieving information about the post, including a link to the MP3 file stored in an Amazon S3 bucket. These methods are exposed as RESTful web services through Amazon API Gateway.

### Features
 - Convert text to speech using Amazon Polly
 - Store post information and MP3 file URLs in Amazon DynamoDB
 - Host a static webpage on Amazon S3
 - Use AWS Lambda functions to process text and retrieve post data
 - Utilize Amazon SNS for decoupling and processing post notifications

### Architecture

![architecture](https://github.com/ghwallis/aws-polly-texttospeech/assets/36977382/cb9000ca-d5e9-4a34-ad65-70fee983d0b7)

### New Post Creation Process


1. **Static Webpage:** Hosted on Amazon S3, sends information about new posts to the RESTful web service.
2. **Amazon API Gateway:** Exposes the RESTful web service and triggers the New Post Lambda function.
3. **New Post Lambda Function:** Inserts new post data into DynamoDB and publishes a message to SNS.
4. **Amazon SNS:** Decouples the process and triggers the Convert to Audio Lambda function.
5. **Convert to Audio Lambda Function:** Converts text to MP3 using Amazon Polly, stores the MP3 file in S3, and updates the DynamoDB table.



### Retrieving Post Information Process
1. Static Webpage: Hosted on Amazon S3, requests post information from the RESTful web service.
2. Amazon API Gateway: Exposes the method to retrieve post information and invokes the Get Post Lambda function.
3. Get Post Lambda Function: Retrieves post information from DynamoDB and returns the data.


### Prerequisites
 - AWS Account
 - Basic knowledge of AWS services (S3, Lambda, DynamoDB, API Gateway, SNS, Polly)
 - Python 3.8+
 - Boto3 (AWS SDK for Python)

## Steps

### Task 1 : Create a DynamoDB table
 - From the AWS Management Console, in the search bar, search for and choose DynamoDB.
 - Choose Create Table
 - Create a new DynamoDB table with:
       - Table name: posts
       - Partition key: id
 - Leave everything else as default and Choose Create Table

### Task 2 : Create an Amazon S3 bucket
 - Select S3 from the AWS Management Console and hit Create bucket.
 - Bucket name: audioposts-NUMBER. Feel free to replace NUMBER with any random number of you choosing. Leave everything else as default and Choose Create Bucket

### Task 3 : Create an SNS topic
- Select SNS from the AWS Management Console and hit Create topic.
- Type: Choose Standard , Name: new_posts , Display name: New Posts. Hit Create Topic.
- Copy the Topic ARN and paste it into a text editor for later use

### Task 4 : Create a new post Lambda function
- Choose Lambda from the AWS Management Console and hit Create function.
- Choose Author from scratch and use the following settings:
     1. Function name: PostReader_NewPost
     2. Runtime: Python 3.12
     3. Expand Change default execution role
     4. Execution role: Choose Use an existing role
     5. Existing role: Choose Lab-Lambda-Role
- Choose Create function.
- Delete the existing code and paste the following code:
   ```
   import boto3
   import os
   import uuid

   def lambda_handler(event, context):

    recordId = str(uuid.uuid4())
    voice = event["voice"]
    text = event["text"]

    print('Generating new DynamoDB record, with ID: ' + recordId)
    print('Input Text: ' + text)
    print('Selected voice: ' + voice)

    # Creating new record in DynamoDB table
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DB_TABLE_NAME'])
    table.put_item(
        Item={
            'id' : recordId,
            'text' : text,
            'voice' : voice,
            'status' : 'PROCESSING'
        }
    )

    # Sending notification about new post to SNS
    client = boto3.client('sns')
    client.publish(
        TopicArn = os.environ['SNS_TOPIC'],
        Message = recordId
    )

    return recordId
   ```
 - Choose Deploy
 - Choose the Configuration tab to configure the environment variables.
 - In the left navigation pane, choose Environment variables.
 - In the Environment variables section, choose Add environment variable.
      1. Key: Enter SNS_TOPIC
      2. Value: Paste the SNS topic you copied earlier
 - Choose Add environment variable again.
      1. Key: Enter DB_TABLE_NAM
      2. Value: Enter posts and Save
### Task 5: Create a convert to audio Lambda function
 - Choose Functions in the top-left navigation pane.
 - Choose Create function.
 - Choose Author from scratch and use the following settings:
      1. Function name: ConvertToAudio
      2. Runtime: Python 3.12
      3. Expand Change default execution role
      4. Execution role: Choose Use an existing role
      5. Existing role: Choose Lab-Lambda-Role
 - Scroll down and choose Create function.
 - Command: Delete the existing code and paste the following code:

```
import boto3
import os
from contextlib import closing
from boto3.dynamodb.conditions import Key, Attr

def lambda_handler(event, context):

    postId = event["Records"][0]["Sns"]["Message"]

    print ("Text to Speech function. Post ID in DynamoDB: " + postId)

    # Retrieving information about the post from DynamoDB table
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DB_TABLE_NAME'])
    postItem = table.query(
        KeyConditionExpression=Key('id').eq(postId)
    )


    text = postItem["Items"][0]["text"]
    voice = postItem["Items"][0]["voice"]

    rest = text

    # Because single invocation of the polly synthesize_speech api can
    # transform text with about 3000 characters, we are dividing the
    # post into blocks of approximately 2500 characters.
    textBlocks = []
    while (len(rest) > 2600):
        begin = 0
        end = rest.find(".", 2500)

        if (end == -1):
            end = rest.find(" ", 2500)

        textBlock = rest[begin:end]
        rest = rest[end:]
        textBlocks.append(textBlock)
    textBlocks.append(rest)

    # For each block, invoke Polly API, which transforms text into audio
    polly = boto3.client('polly')
    for textBlock in textBlocks:
        response = polly.synthesize_speech(
            OutputFormat='mp3',
            Text = textBlock,
            VoiceId = voice
        )

        # Save the audio stream returned by Amazon Polly on Lambda's temp
        # directory. If there are multiple text blocks, the audio stream
        # is combined into a single file.
        if "AudioStream" in response:
            with closing(response["AudioStream"]) as stream:
                output = os.path.join("/tmp/", postId)
                with open(output, "wb") as file:
                    file.write(stream.read())

    s3 = boto3.client('s3')
    s3.upload_file('/tmp/' + postId,
      os.environ['BUCKET_NAME'],
      postId + ".mp3")
    s3.put_object_acl(ACL='public-read',
      Bucket=os.environ['BUCKET_NAME'],
      Key= postId + ".mp3")

    location = s3.get_bucket_location(Bucket=os.environ['BUCKET_NAME'])
    region = location['LocationConstraint']

    if region is None:
        url_beginning = "https://s3.amazonaws.com/"
    else:
        url_beginning = "https://s3-" + str(region) + ".amazonaws.com/"

    url = url_beginning \
            + str(os.environ['BUCKET_NAME']) \
            + "/" \
            + str(postId) \
            + ".mp3"

    # Updating the item in DynamoDB
    response = table.update_item(
        Key={'id':postId},
          UpdateExpression=
            "SET #statusAtt = :statusValue, #urlAtt = :urlValue",
          ExpressionAttributeValues=
            {':statusValue': 'UPDATED', ':urlValue': url},
        ExpressionAttributeNames=
          {'#statusAtt': 'status', '#urlAtt': 'url'},
    )

    return
```



### Video Demo of Final Product.

Converting provided text to speech and retrieving mp3 audio from DynamoDB.

<video src='https://github.com/ghwallis/aws-polly-texttospeech/assets/36977382/1da7d153-d588-47db-814c-b41b2ab5d942' width=180/>
