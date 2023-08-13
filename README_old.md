# TechConf Registration Website

## Project Overview
The TechConf website allows attendees to register for an upcoming conference. Administrators can also view the list of attendees and notify all attendees via a personalized email message.

The application is currently working but the following pain points have triggered the need for migration to Azure:
 - The web application is not scalable to handle user load at peak
 - When the admin sends out notifications, it's currently taking a long time because it's looping through all attendees, resulting in some HTTP timeout exceptions
 - The current architecture is not cost-effective 

In this project, you are tasked to do the following:
- Migrate and deploy the pre-existing web app to an Azure App Service
- Migrate a PostgreSQL database backup to an Azure Postgres database instance
- Refactor the notification logic to an Azure Function via a service bus queue message

## Dependencies

You will need to install the following locally:
- [Postgres](https://www.postgresql.org/download/)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Azure Function tools V3](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack)

## Project Instructions

### Part 1: Create Azure Resources and Deploy Web App
1. Create a Resource group
2. Create an Azure Postgres Database single server
   - Add a new database `techconfdb`
   - Allow all IPs to connect to database server
   - Restore the database with the backup located in the data folder
3. Create a Service Bus resource with a `notificationqueue` that will be used to communicate between the web and the function
   - Open the web folder and update the following in the `config.py` file
      - `POSTGRES_URL`
      - `POSTGRES_USER`
      - `POSTGRES_PW`
      - `POSTGRES_DB`
      - `SERVICE_BUS_CONNECTION_STRING`
4. Create App Service plan
5. Create a storage account
6. Deploy the web app

### Part 2: Create and Publish Azure Function
1. Create an Azure Function in the `function` folder that is triggered by the service bus queue created in Part 1.

      **Note**: Skeleton code has been provided in the **README** file located in the `function` folder. You will need to copy/paste this code into the `__init.py__` file in the `function` folder.
      - The Azure Function should do the following:
         - Process the message which is the `notification_id`
         - Query the database using `psycopg2` library for the given notification to retrieve the subject and message
         - Query the database to retrieve a list of attendees (**email** and **first name**)
         - Loop through each attendee and send a personalized subject message
         - After the notification, update the notification status with the total number of attendees notified
2. Publish the Azure Function

### Part 3: Refactor `routes.py`
1. Refactor the post logic in `web/app/routes.py -> notification()` using servicebus `queue_client`:
   - The notification method on POST should save the notification object and queue the notification id for the function to pick it up
2. Re-deploy the web app to publish changes

## Monthly Cost Analysis (Development):
Complete a month cost analysis of each Azure resource to give an estimate total cost using the table below:

| Azure Resource | Service Tier | Monthly Cost |
| ------------ | ------------ | ------------ |
| *Azure Postgres Database* |  Basic, 1 vCore(s), 5 GB   |  25.32 USD   |
| *Azure Service Bus*   |    Basic     |   0.05 USD   |
| *Web App*  |   F1      |     Free       |
| *Function App*  |    Consumption     |   Pay only for the number of executions, execution time, and memory used. |
| *SendGrid for Azure*  |    Free     |   0 USD |

## Monthly Cost Analysis (In Production):
| Azure Resource | Service Tier | Monthly Cost |
| ------------ | ------------ | ------------ |
| *Azure Postgres Database* |  Basic, 1 vCore(s), 5 GB   |  25.32 USD   |
| *Azure Service Bus*   |    Basic     |   0.05 USD   |
| *Web App*  |   Basic B1      |     13.87 USD       |
| *Function App*  |    Consumption     |   < 10 USD |
| *SendGrid for Azure*  |    Essentials 50K     |   19.95 USD|
| *Total*  |         |   < $70  |

## Architecture Explanation
Here is the explanation and reasoning for my architecture selection for both the Azure Web App and Azure Function:

In the previous architecture, the web app was directly connected to the database and the notification was triggered from the web app itself. This architecture is not scalable to handle user load at peak. Timeout issues can easily happen.

For example:
If there are 1000 attendees the user must wait on the notification page.
Until all attendees are notified.

So in order to decouple the app into frontend and functions, we have used the Azure service bus queue to trigger the notification function.
The web app triggers the queue and the function fetches the notification id from the queue and processes the mail triggering function.

This app is not very big so i chose to deploy it in Azure.

Azure Web, Azure Function support free tier so i can test in development easily.
And in production they can scale up/down easily to fit on the trafic of my app.

Cost-effectiveness: pay less, instead of paying for the total app, only pay for each component

For Azure Service Bus I chose Basic Tier because 1 million operations per month is enough for our app and it is the cheapest option.

For Function App I chose Consumption Tier because it is the cheapest option and it is pay as you go. So we only pay for the number of executions, execution time, and memory used. And in production we can scale up/down easily to fit on the trafic of our app.

For Web App I chose Basic B1 Tier because it is the cheapest option and it is free for the first 12 months. And in production we can scale up/down easily to fit on the trafic of our app. 100 total ACU and 1.75 GB memory is enough for our app. 0.019 USD cost per hour or
13.87 USD cost per month is very cheap but it is enough for our app.

For sending mails in production, i will use the pack Essentials 50K
Because it supports up to 50,000 emails per month and in my company which would be having under 10k employees and under 5 conferences per month.
My app will send less than 50k emails per month so it is enough for my app.

## Link for the web app:
https://techconf-999-wa.azurewebsites.net
