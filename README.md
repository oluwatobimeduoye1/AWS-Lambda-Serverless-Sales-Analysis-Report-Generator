# Overview
This project demonstrates the deployment and configuration of an AWS Lambda-based serverless solution for generating daily sales analysis reports. The Lambda function retrieves sales data from a MySQL database running on an Amazon EC2 LAMP (Linux, Apache, MySQL, PHP) stack instance, processes the data, and emails the analysis report.
## Key Components
- AWS Lambda — Executes a scheduled function to generate and email the sales report.

- Amazon EC2 (LAMP Stack) — Hosts the MySQL database containing sales data.

- AWS Systems Manager Parameter Store — Securely stores and manages database connection parameters.

- Amazon Simple Notification Service (SNS) — Sends the sales analysis report via email.


#### The following diagram shows the architecture of the sales analysis report solution and illustrates the order in which actions occur.

<img width="1588" height="850" alt="image" src="https://github.com/user-attachments/assets/e7305e4f-c594-414f-a597-bb8eba5349be" />

The diagram includes the following function steps:

##### Step	Details
1.	An Amazon CloudWatch Events event calls the salesAnalysisReport Lambda function at 8 PM every day Monday through Saturday.
2.	The salesAnalysisReport Lambda function invokes another Lambda function, salesAnalysisReportDataExtractor, to retrieve the report data.
3.	The salesAnalysisReportDataExtractor function runs an analytical query against the café database (cafe_db).
4.	The query result is returned to the salesAnalysisReport function.
5.	The salesAnalysisReport function formats the report into a message and publishes it to the salesAnalysisReportTopic Amazon Simple Notification Service (Amazon SNS) topic.
6.	The salesAnalysisReportTopic SNS topic sends the message by email to the administrator.

## Objectives
This project demonstrates how to build a serverless reporting solution on AWS by configuring IAM permissions, creating Lambda layers for external dependencies, developing Lambda functions for data extraction and report delivery, setting up scheduled invocations and function chaining, and utilizing Amazon CloudWatch Logs for monitoring and troubleshooting.


# First Create an salesAnalysisReport & salesAnalysisReportDERole  IAM role with Trust relationship 
- Create salesAnalysisReport IAM role with the following permission:

<img width="1354" height="416" alt="Screenshot 2025-07-18 213646" src="https://github.com/user-attachments/assets/ace2370a-c0a9-4986-b781-6772cdd9c9e8" />

   i.AmazonSNSFullAccess provides full access to Amazon SNS resources.

   ii AmazonSSMReadOnlyAccess provides read-only access to Systems Manager resources.

   iii AWSLambdaBasicRunRole provides write permissions to CloudWatch logs (which are required by every Lambda function).

   iv. AWSLambdaRole gives a Lambda function the ability to invoke another Lambda function. 

- Then create a Trust relationship that allows the Lambda service use the salesAnalysisReport IAM role
<img width="654" height="528" alt="image" src="https://github.com/user-attachments/assets/4e18d923-6e77-445d-b344-5faa285bd357" />

The salesAnalysisReport Lambda function that will be created later will uses the salesAnalysisReportRole role

- Create salesAnalysisReportDERole IAM role with the following permission:

  <img width="1358" height="508" alt="Screenshot 2025-07-18 214000" src="https://github.com/user-attachments/assets/485d7c37-fd11-4080-a71e-a39df3b3b9e3" />

     i.  AWSLambdaBasicRunRole provides write permissions to CloudWatch logs.

     ii. AWSLambdaVPCAccessRunRole provides permissions to manage elastic network interfaces to connect a function to a virtual private cloud (VPC).

- Then  create a Trust relationship that allows the Lambda service use the salesAnalysisReportDERole IAM role

<img width="1364" height="501" alt="Screenshot 2025-07-18 214247" src="https://github.com/user-attachments/assets/f52fc8fb-9ae1-485e-8246-046a9bef94bf" />

The salesAnalysisReportDataExtractor Lambda function that you create next uses the salesAnalysisReportDERole role.


# Next, Create a a Lambda layer and a data extractor Lambda function
first create a Lambda layer, and then create a Lambda function that uses the layer.

- The salesAnalysisReportDataExtractor-v3.zip file is a Python implementation of a Lambda function that makes use of the PyMySQL open-source client library to access the MySQL café database. The library has been packaged as pymysql-v3.zip and will be uploaded as a Lambda layer.

#### Creating a Lambda Layer
we create a Lambda layer named pymysqlLibrary and upload the client library into it so that it can be used by any function that requires it. Lambda layers provide a flexible mechanism to reuse code between functions so that the code does not have to be included in each function’s deployment package.

- In the AWS Management Console, choose Services > Compute > Lambda.
- Choose Layers.
- Choose Create layer.
- Configure the following layer settings:
       i. Name, 

      ii. Description, 

     iii. Select Upload a .zip file. To upload the pymysql-v3.zip file, 

      iv. Compatible runtimes, choose Python 3.9.

- then create 
<img width="1350" height="539" alt="Screenshot 2025-07-18 225139" src="https://github.com/user-attachments/assets/d375631b-a2f0-445b-9229-3725a66a6bb9" />

<img width="1354" height="522" alt="Screenshot 2025-07-18 231215" src="https://github.com/user-attachments/assets/ef68c50d-92c3-41b3-9b18-b84d1c1a7a9d" />


#### Creating a data extractor Lambda function
- In the navigation pane, choose Functions to open the Functions dashboard page.

- Choose Create function, and configure the following options:

     i  At the top of the Create function page, select Author from scratch.

     ii For Function name, enter salesAnalysisReportDataExtractor

     iii For Runtime, choose Python 3.9.

     iv. Expand Change default execution role, and configure the following options:    

          For Execution role, choose Use an existing role.

          For Existing role:, choose salesAnalysisReportDERole.

      v. Create function.
<img width="1366" height="500" alt="Screenshot 2025-07-18 232350" src="https://github.com/user-attachments/assets/7d02f9ba-629c-4d2f-a156-a3c05a108b61" />

<img width="1366" height="304" alt="Screenshot 2025-07-18 233237" src="https://github.com/user-attachments/assets/5510ec02-288a-42e7-a1fc-9b6ece50d61e" />

## Add the Lambda layer to the function
- In the Function overview panel, choose Layers.

- At the bottom of the page, in the Layers panel, choose Add a layer.

- On the Add layer page, configure the following options:

i Choose a layer: Choose Custom layers.

ii Custom layers: Choose pymysqlLibrary.

iii Version: Choose 2.

Choose Add.

The Function overview panel shows a count of (1) in the Layers node for the function.
<img width="1366" height="479" alt="Screenshot 2025-07-19 000004" src="https://github.com/user-attachments/assets/3e9c53c3-46be-4fa3-a7ca-7ff9a74fb1e2" />
<img width="1359" height="534" alt="Screenshot 2025-07-19 000225" src="https://github.com/user-attachments/assets/f80fb9af-6e39-4278-b079-062875fcb238" />

The Function overview panel shows a count of (1) in the Layers node for the function.

#### Importing the code for the data extractor Lambda function
- Go to the Lambda > Functions > salesAnalysisReportDataExtractor page.

- In the Runtime settings panel, choose Edit.

    i For Handler, enter salesAnalysisReportDataExtractor.lambda_handler

- Choose Save.

- In the Code source panel, choose Upload from.

- Choose .zip file.

- Choose Upload, and then navigate to and select the salesAnalysisReportDataExtractor-v3.zip file that you downloaded earlier.

Choose Save.
<img width="1359" height="534" alt="Screenshot 2025-07-19 000225" src="https://github.com/user-attachments/assets/abe44b2f-4deb-41a6-9e01-6a269af0f2bf" />

<img width="1365" height="541" alt="Screenshot 2025-07-19 002848" src="https://github.com/user-attachments/assets/485f6006-cba4-41ee-a635-852981bac0ad" />

#### Configuring network settings for the function
The final step before testing the function is to configure its network settings. As the architecture diagram at the start of this project shows, this function requires network access to the café database, which runs in an EC2 LAMP instance. Therefore, you need to specify the instance’s VPC, subnet, and security group information in the function’s configuration.
- Choose the Configuration tab, and then choose VPC.

####### Choose Edit, and configure the following options:

i. VPC: Choose the option with Cafe VPC as the Name.

ii Subnets: Choose the option with Cafe Public Subnet 1 as the Name.
iii. Security groups: Choose the option with CafeSecurityGroup as the Name.

<img width="1366" height="539" alt="Screenshot 2025-07-19 011234" src="https://github.com/user-attachments/assets/51db505a-a30d-48a0-947d-d2974fe46d48" /> <img width="1366" height="543" alt="Screenshot 2025-07-19 011415" src="https://github.com/user-attachments/assets/1962bbc6-e5f9-43bb-8019-1289c98c99f1" />


# Testing the data extractor Lambda function
We are now ready to test the salesAnalysisReportDataExtractor function. To invoke it, we need to supply values for the café database connection parameters. Recall that these are stored in Parameter Store.

- On a new browser tab, open the AWS Management Console, and choose Services > Management & Governance > Systems Manager.

- In the navigation pane, choose Parameter Store.
<img width="1364" height="541" alt="Screenshot 2025-07-19 013616" src="https://github.com/user-attachments/assets/b538e9b0-a03a-4ce3-9385-701034c4bfd6" />


- Choose each of the following parameter names, and copy and paste the Value of each one into a text editor document:

/cafe/dbUrl

/cafe/dbName

/cafe/dbUser

/cafe/dbPassword

- Return to the Lambda Management Console browser tab. On the salesAnalysisReportDataExtractor function page, choose the Test tab.

- Configure the Test event panel as follows:

For Test event action, select Create new event.

For Event name, enter SARDETestEvent

For Template, choose hello-world.

- In the Event JSON pane, replace the JSON object with the following JSON object:

```
{
  "dbUrl": "<value of /cafe/dbUrl parameter>",
  "dbName": "<value of /cafe/dbName parameter>",
  "dbUser": "<value of /cafe/dbUser parameter>",
  "dbPassword": "<value of /cafe/dbPassword parameter>"
}
```
<img width="1366" height="454" alt="Screenshot 2025-07-19 020028" src="https://github.com/user-attachments/assets/87af1824-bcfc-4b12-8eb6-ba40cf3914c8" />

In this code, substitute the value of each parameter with the values that you pasted into a text editor in the previous steps. Enclose these values in quotation marks.

Choose Save.

Choose Test.
<img width="1360" height="523" alt="Screenshot 2025-07-19 014509" src="https://github.com/user-attachments/assets/de2876ca-6ed9-45fa-ae58-e1ae6cb7839b" />


After a few moments, the page shows the message "Execution result: failed". 



#### Troubleshooting the data extractor Lambda function
In the Execution result pane, choose Details shows:

<img width="633" height="126" alt="Screenshot 2025-07-19 020922" src="https://github.com/user-attachments/assets/ae25f3d3-f6eb-49cf-a934-fbc01316ff97" />

This message indicates that the function timed out after 3 seconds.

The Log output section includes lines starting with the following keywords:

- START indicates that the function started running.
- END indicates that the function finished running.
- REPORT provides a summary of the performance and resource utilization statistics related to when the function ran.

#### Analyzing and correcting the Lambda function
Here are a few hints to help you find the solution:

- One of the first things that this function does is connect to the MySQL database running in a separate EC2 instance. It waits a certain amount of time to establish a successful connection. After this time passes, if the connection is unsuccessful, the function times out.

- By default, a MySQL database uses the MySQL protocol and listens on port number 3306 for client access. However port wasn't configured in security group
<img width="1363" height="454" alt="Screenshot 2025-07-19 021958" src="https://github.com/user-attachments/assets/165f46ac-5a3b-41c1-9fdf-aa1e9fea0a9f" />
Once you have corrected the problem, test was re-run and successful
<img width="1361" height="466" alt="Screenshot 2025-07-19 022107" src="https://github.com/user-attachments/assets/f8ab1f76-b44c-41b8-b167-bec6e33c5e3b" /> <img width="1366" height="542" alt="Screenshot 2025-07-19 022135" src="https://github.com/user-attachments/assets/3d80d405-e875-4487-90f7-40a6fd8af4ee" /> <img width="1364" height="519" alt="Screenshot 2025-07-19 022753" src="https://github.com/user-attachments/assets/db5cb4fc-cc98-477e-b215-84908745b99f" />
# Placing an order and testing again
open cafe website and make order to test the website 
<img width="1354" height="694" alt="Screenshot 2025-07-19 024058" src="https://github.com/user-attachments/assets/fb5ddf13-9a37-44ad-a03b-b2d210eb334d" />

<img width="1314" height="704" alt="Screenshot 2025-07-19 024617" src="https://github.com/user-attachments/assets/0de42d2d-41a5-465f-bc09-f6750cc88647" /> <img width="1293" height="496" alt="Screenshot 2025-07-19 024701" src="https://github.com/user-attachments/assets/8800a453-1ca1-4cbf-87ba-b09a5cda65fc" />

salesAnalysisReportDataExtractor function page and run a test
The returned JSON object now contains product quantity information in the body field similar to the order made on the website. confirming the salesAnalysisReportDataExtractor Lambda function !!!

<img width="656" height="516" alt="Screenshot 2025-07-19 025006" src="https://github.com/user-attachments/assets/1eabf1ca-61ae-476e-ae0a-ca8ec88862db" />

# Configuring notifications
- Create an SNS topic
    Protocol: Choose Email.

Endpoint: Enter an email address that you can access.

<img width="653" height="490" alt="Screenshot 2025-07-19 030019" src="https://github.com/user-attachments/assets/46baf9cf-a1ee-4da0-b843-06391e4fb0bf" />

- Choose Create subscription.

The subscription is created and has a Status of Pending confirmation.

Check the inbox for the email address that you provided.

<img width="669" height="279" alt="Screenshot 2025-07-19 030221" src="https://github.com/user-attachments/assets/6ff0260a-f630-405d-a565-dd1796ad2ecf" />

You should see an email from SARTopic with the subject "AWS Notification - Subscription Confirmation."

Open the email, and choose Confirm subscription.

- A new browser tab opens and displays a page with the message "Subscription confirmed!"
<img width="670" height="525" alt="Screenshot 2025-07-19 030435" src="https://github.com/user-attachments/assets/e5fbfce8-5ad1-40a2-beb1-a6cd3abdc3da" />

# Creating the salesAnalysisReport Lambda function

Next, we create and configure the salesAnalysisReport Lambda function. This function is the main driver of the sales analysis report flow. It does the following:

- Retrieves the database connection information from Parameter Store

- Invokes the salesAnalysisReportDataExtractor Lambda function, which retrieves the report data from the database

- Formats and publishes a message containing the report data to the SNS topic

#### create function through ec2 instance CLI

  <img width="669" height="440" alt="Screenshot 2025-07-19 032144" src="https://github.com/user-attachments/assets/226d5c6e-96ce-4887-bbfb-dd708faad61d" />

Next, you use the Lambda create-function command to create the Lambda function and configure it to use the salesAnalysisReportRole IAM role

```
aws lambda create-function \
  --function-name salesAnalysisReport \
  --runtime python3.9 \
  --zip-file fileb://salesAnalysisReport-v2.zip \
  --handler salesAnalysisReport.lambda_handler \
  --region us-west-2 \
  --role arn:aws:iam::803020999895:role/salesAnalysisReportRole
```
<img width="659" height="475" alt="Screenshot 2025-07-19 032919" src="https://github.com/user-attachments/assets/015dd75e-78bb-43a6-8c82-d2da1320e544" />

Once the command completes, it returns a JSON object describing the attributes of the function. You now complete its configuration and unit test it.

#### Next, Configure the salesAnalysisReport Lambda function

- Open the Lambda management console.

- Choose Functions, and then choose salesAnalysisReport
- Choose the Configuration tab, and choose Environment variables.

<img width="661" height="482" alt="Screenshot 2025-07-19 033634" src="https://github.com/user-attachments/assets/7925f235-1a4d-42e3-b281-a26208297713" />

- Choose Edit.

- Choose Add environment variable, and configure the following options:

Key: Enter topicARN

Value: Paste the ARN value of the salesAnalysisReportTopic SNS topic 

Choose Save.

# Testing the salesAnalysisReport Lambda function

 Test tab, and configure the test event ,A green box with the message “Execution result: succeeded (logs)” appears.
<img width="678" height="525" alt="Screenshot 2025-07-19 034250" src="https://github.com/user-attachments/assets/1169f2cc-c3db-4bd6-bc50-9cb01423fa27" />

An email from AWS Notifications with the subject "Daily Sales Analysis Report." will be automatical sent to the subscribed email with details of orders placed on the cafe website 
<img width="1009" height="372" alt="Screenshot 2025-07-19 034520" src="https://github.com/user-attachments/assets/837d634b-0b8c-40c1-818f-e41a4d5efae6" />

#### Adding a trigger to the salesAnalysisReport Lambda function

To complete the implementation of the salesAnalysisReport function, configure the report to be initiated Monday through Saturday at 8 PM each day. To do so, use a CloudWatch Events event as the trigger mechanism
Function overview panel, choose Add trigger. The Add trigger panel is displayed.

In the Add trigger panel, configure the following options:

In the Trigger configuration pane, from the dropdown list, choose EventBridge (CloudWatch Events).

For Rule, choose Create a new rule. 

For Rule name, enter salesAnalysisReportDailyTrigger

For Rule description, enter Initiates report generation on a daily basis

For Rule type, choose Schedule expression.

For Schedule expression

<img width="1347" height="518" alt="Screenshot 2025-07-19 035706" src="https://github.com/user-attachments/assets/badca315-a802-4d4e-8217-003856e5b578" />
<img width="673" height="506" alt="Screenshot 2025-07-19 035742" src="https://github.com/user-attachments/assets/78894982-b4c3-4ff2-8881-1cfea7aed8f5" />

# Use Case and Benefits
This project demonstrates a real-world serverless reporting solution for automating daily business insights using AWS services. It simulates a café's sales reporting system, where data is extracted from a MySQL database, processed, and emailed to stakeholders without manual intervention.

### Use Case:
- Automated Daily Reporting — Automatically generate and distribute daily sales analysis reports based on live transactional data.

- Serverless Architecture — Leverages AWS Lambda, eliminating the need for dedicated servers or manual scheduling scripts.

- Secure Data Access — Uses AWS Systems Manager Parameter Store for secure, managed storage of database credentials.

- Event-Driven Integration — Connects AWS services (Lambda, EC2, SNS, CloudWatch) in a seamless, event-triggered workflow.

- Scalable and Maintainable — Modular Lambda functions with shared layers promote reuse and easy maintenance.

### Business Benefits:
- Operational Efficiency — Reduces manual reporting tasks, saving time and improving consistency.

- Cost-Effective — Pay-as-you-go serverless model ensures cost control without provisioning unnecessary infrastructure.

- Improved Visibility — Timely reports help business managers make informed decisions based on up-to-date sales data.

- Enhanced Security — Centralized credential management and fine-grained IAM roles ensure compliance and data protection.

- Flexibility — Easily adaptable for other reporting, monitoring, or notification use cases across different business domains.



