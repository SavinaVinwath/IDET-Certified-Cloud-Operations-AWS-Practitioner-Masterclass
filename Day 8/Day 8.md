# DAY 8 - Serverless, SNS, Monitoring & Alarms

---

## Topic 7A: Serverless & AWS Lambda

### What is Serverless?

Traditional approach: You rent a server (EC2), install your OS, patch it, scale it, pay even when idle.

**Serverless approach:** You only write code. AWS handles everything else - servers, OS, scaling, availability. You pay **only when your code runs**, down to the millisecond.

### Event-Driven Architecture

In serverless, code doesn't run constantly - it runs **in response to events**:

| Event Source    | Example                                       |
| --------------- | --------------------------------------------- |
| HTTP Request    | User hits an API endpoint                     |
| File Upload     | Image uploaded to S3                          |
| Database Change | New row inserted in DynamoDB                  |
| Schedule        | Run every night at midnight (like a cron job) |
| Message Queue   | Message arrives in SQS                        |

### AWS Lambda

- A **Function-as-a-Service (FaaS)** offering
- Supports Python, Node.js, Java, Go, .NET, Ruby
- Free Tier: **1 million requests/month** and **400,000 GB-seconds** of compute per month - very generous for learning

### Amazon API Gateway

- A fully managed service that creates **HTTP/REST/WebSocket APIs**
- Acts as the "front door" for your Lambda function
- When someone calls a URL, API Gateway receives the request and **triggers your Lambda**

```
User → [URL] → API Gateway → Lambda Function → Response → User
```

---

## DEMO 7: Deploy a Python Lambda Function with API Gateway

**Goal:** Create a Lambda function in Python that returns a JSON response, then expose it via a public API Gateway URL.

---

### Step 1 - Create the Lambda Function

1. In the AWS Console, search for **Lambda** in the top search bar and click it
2. Click the orange **"Create function"** button
3. Select **"Author from scratch"**
4. Fill in the following:
   - **Function name:** `kt-hello-api`
   - **Runtime:** `Python 3.12`
   - **Architecture:** `x86_64`
5. Expand **"Change default execution role"** → Leave it as "Create a new role with basic Lambda permissions"
6. Click the orange **"Create function"** button at the bottom

---

### Step 2 - Write the Python Code

1. You'll see a built-in code editor. Click on the file `lambda_function.py`
2. **Select all** the existing code (Ctrl+A / Cmd+A) and **delete it**
3. Paste in the following code exactly:

```python
import json

def lambda_handler(event, context):
    # Get the HTTP method and path if called from API Gateway
    http_method = event.get('requestContext', {}).get('http', {}).get('method', 'DIRECT')
    path = event.get('requestContext', {}).get('http', {}).get('path', '/test')

    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'message': 'Hello from AWS Lambda!',
            'status': 'success',
            'method': http_method,
            'path': path,
            'info': 'This response was generated serverlessly - no server was managed!'
        }, indent=2)
    }
```

4. Click **"Deploy"** (the grey button above the code editor)
5. Wait for the green banner: _"Successfully updated the function kt-hello-api"_

---

### Step 3 - Test the Lambda Function Directly

1. Click the **"Test"** tab (next to Code tab)
2. Select **"Create new event"**
3. **Event name:** `my-test-event`
4. Leave the default JSON as-is (`{ "key1": "value1" ... }`)
5. Click **"Save"**
6. Click the orange **"Test"** button
7. Expand the **"Response"** section - you should see:

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\n  \"message\": \"Hello from AWS Lambda!\", ..."
}
```

**Lambda works!** Now let's give it a public URL via API Gateway.

---

### Step 4 - Add API Gateway Trigger

1. Click the **"Configuration"** tab
2. Click **"Triggers"** in the left sidebar
3. Click **"Add trigger"**
4. In the dropdown, select **"API Gateway"**
5. Select **"Create a new API"**
6. **API type:** `HTTP API`
7. **Security:** `Open` (for this demo - in production you'd add auth)
8. Click **"Add"**

---

### Step 5 - Test the Public URL

1. Click the **"Configuration"** tab → **"Triggers"**
2. You'll see the API Gateway trigger with an **API endpoint URL** - it looks like:
   `https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/default/kt-hello-api`
3. **Copy that URL**
4. Open a new browser tab and **paste the URL** → press Enter
5. You should see the JSON response in your browser!

---

### Step 6 - Show the Execution Logs

1. Go back to Lambda → click the **"Monitor"** tab
2. Click **"View CloudWatch logs"**
3. Click the latest log stream
4. Show the team the START, END, and REPORT lines - these show execution time and memory used
5. Point out: **"Duration: X ms"** - you only paid for those milliseconds

---

## Topic 8: Security & Monitoring

### The Shared Responsibility Model

This is AWS's most important concept for security:

```
┌─────────────────────────────────────────────┐
│             YOUR RESPONSIBILITY             │
│  - Your application code                   │
│  - Data you store (encryption choices)     │
│  - IAM users, roles, permissions           │
│  - OS patches (if you use EC2)             │
│  - Security group / firewall rules         │
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│             AWS'S RESPONSIBILITY            │
│  - Physical data center security           │
│  - Hardware (servers, network, storage)    │
│  - Hypervisor (virtualization layer)       │
│  - Managed service infrastructure          │
│    (e.g., Lambda runtime, RDS engine)      │
└─────────────────────────────────────────────┘
```

### Monitoring Tools

| Service        | What it Does                                                                                                |
| -------------- | ----------------------------------------------------------------------------------------------------------- |
| **CloudWatch** | Collects metrics (CPU, memory, request counts), logs, and lets you set alarms. The "health monitor" of AWS. |
| **CloudTrail** | Records every API call made in your account - who did what, when, from where. The "security camera" of AWS. |

### Security Protection Services

| Service                            | What it Protects Against                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------------------------------- |
| **WAF** (Web Application Firewall) | SQL injection, XSS, bad bots, malicious IPs - Layer 7 attacks                                     |
| **Shield**                         | DDoS (Distributed Denial of Service) attacks. Shield Standard is free for all accounts.           |
| **GuardDuty**                      | AI-powered threat detection - finds unusual behaviour like crypto mining, compromised credentials |

---

## DEMO 8: CloudWatch Dashboard + SNS Email Alarm

**Goal:** Build a CloudWatch Dashboard, create an SNS topic that sends email alerts, and wire up a CPU alarm.

---

### Part A - Create a CloudWatch Dashboard (5 min)

1. In the AWS Console, search for **CloudWatch** and open it
2. In the left sidebar, click **"Dashboards"**
3. Click **"Create dashboard"**
4. **Dashboard name:** `KT-Monitoring-Dashboard`
5. Click **"Create dashboard"**
6. A popup asks what widget to add - select **"Line"** → Click **"Next"**
7. Select **"Metrics"** as the data source → Click **"Next"**
8. In the metrics browser, click **"Lambda"** → Click **"By Function Name"**
9. Find your function `kt-hello-api` and tick the checkbox next to **"Invocations"**
10. Click **"Create widget"**
11. Click **"Save"** (top right)

You have a live dashboard showing Lambda invocations!

---

### Part B - Create an SNS Topic for Email Alerts

SNS (Simple Notification Service) is AWS's messaging service. We'll use it to send email alerts.

1. Search for **SNS** in the AWS Console and open it
2. In the left sidebar, click **"Topics"**
3. Click **"Create topic"**
4. **Type:** Select `Standard`
5. **Name:** `kt-alerts-topic`
6. Scroll down and click **"Create topic"**

Now create a subscription (to receive emails):

7. On the topic page, click **"Create subscription"**
8. **Protocol:** `Email`
9. **Endpoint:** Enter **your real email address**
10. Click **"Create subscription"**
11. **Check your email inbox** - you'll get a confirmation email from AWS
12. Click **"Confirm subscription"** in that email
13. Go back to the AWS Console - the subscription status should change from `PendingConfirmation` to `Confirmed`

---

### Part C - Create a CloudWatch Alarm for Lambda Errors

We'll create an alarm that fires if your Lambda has any errors (easier to trigger than CPU on EC2).

1. In CloudWatch, click **"Alarms"** → **"All alarms"** in the left sidebar
2. Click **"Create alarm"**
3. Click **"Select metric"**
4. Click **"Lambda"** → **"By Function Name"**
5. Find `kt-hello-api` and tick the checkbox next to **"Errors"**
6. Click **"Select metric"**
7. Configure the alarm:
   - **Statistic:** `Sum`
   - **Period:** `1 minute`
8. Under **"Conditions":**
   - **Threshold type:** `Static`
   - **Whenever Errors is...:** `Greater than or equal to`
   - **than:** `1`
9. Click **"Next"**
10. Under **"Notification":**
    - **Alarm state trigger:** `In alarm`
    - **Select an SNS topic:** Choose `kt-alerts-topic`
11. Click **"Next"**
12. **Alarm name:** `kt-lambda-error-alarm`
13. Click **"Next"** → **"Create alarm"**

---

### Part D - Trigger the Alarm

Let's cause a Lambda error on purpose to see the alarm fire:

1. Go back to **Lambda** → `kt-hello-api`
2. Click the **"Code"** tab
3. Add a deliberate error - change line 3 to `raise Exception("Demo error!")`
4. Click **"Deploy"**
5. Click **"Test"** → run the test (it will fail - that's intentional)
6. Go back to **CloudWatch → Alarms**
7. Within 1–2 minutes, the alarm will turn **red (In alarm)**
8. Check your email - you'll receive an SNS alert!
