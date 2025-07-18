# Overview
This project demonstrates the deployment and configuration of an AWS Lambda-based serverless solution for generating daily sales analysis reports. The Lambda function retrieves sales data from a MySQL database running on an Amazon EC2 LAMP (Linux, Apache, MySQL, PHP) stack instance, processes the data, and emails the analysis report.
## Key Components
- AWS Lambda — Executes a scheduled function to generate and email the sales report.

- Amazon EC2 (LAMP Stack) — Hosts the MySQL database containing sales data.

- AWS Systems Manager Parameter Store — Securely stores and manages database connection parameters.

- Amazon Simple Notification Service (SES) — Sends the sales analysis report via email.


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



