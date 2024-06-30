## Text to Voice App using AWS

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
   
### Video Demo of Final Product.

Converting provided text to speech and retrieving mp3 audio from DynamoDB.

<video src='https://github.com/ghwallis/aws-polly-texttospeech/assets/36977382/1da7d153-d588-47db-814c-b41b2ab5d942' width=180/>
