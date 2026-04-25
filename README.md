# Assignment 5: Secure Serverless Architectures
### ZG504 – Security Fundamentals for Cloud | BITS Pilani

> **Objective:** Develop a serverless API-based application with proper security controls using AWS Lambda, API Gateway, and CloudWatch.

---

## Architecture Overview

```
Client (curl / Postman)
        |
        v
  API Gateway (REST API)
        |
        v
  AWS Lambda Function  <---->  CloudWatch Logs
        |
   IAM Execution Role
```

You will build a **secure To-Do API** (or any simple CRUD API) backed by a Lambda function, exposed through API Gateway, secured with IAM execution roles and input validation, and monitored via CloudWatch.

---

## Prerequisites

- Access to AWS Academy Learner Lab (BITS provided)
- Lab must be **started** (green dot) before proceeding
- Services used: **Lambda, API Gateway, CloudWatch, IAM** — all available in the Learner Lab

---

## Step-by-Step Implementation

---

### STEP 1 — Create the IAM Execution Role for Lambda

> **Why:** Lambda needs permission to write logs to CloudWatch. Least privilege means we give it only what it needs.

1. Go to **IAM → Roles → Create role**
2. **Trusted entity:** AWS Service → **Lambda**
3. Click **Next**
4. Search and attach this policy: `AWSLambdaBasicExecutionRole`
5. Click **Next**
6. **Role name:** `SecureLambdaExecutionRole`
7. Click **Create role**

📸 *Screenshot: Role created with AWSLambdaBasicExecutionRole policy attached*

---

### STEP 2 — Create the Lambda Function

1. Go to **Lambda → Create function**
2. Choose **Author from scratch**
3. Fill in:
   - **Function name:** `SecureToDoAPI`
   - **Runtime:** Python 3.12
   - **Architecture:** x86_64
4. Under **Permissions → Change default execution role:**
   - Select **Use an existing role**
   - Choose `SecureLambdaExecutionRole`
5. Click **Create function**

📸 *Screenshot: Lambda function created successfully*

---

### STEP 3 — Write the Lambda Function Code (with Input Validation)

In the Lambda **Code** tab, replace the default code with the following:

```python
import json
import re
import logging

# Set up logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# In-memory store (for demo purposes)
todo_store = {}

def validate_input(body):
    """Input validation — rejects malicious or malformed input."""
    if not body:
        return False, "Request body is empty"
    
    title = body.get("title", "")
    
    # Must not be empty
    if not title or not title.strip():
        return False, "Title cannot be empty"
    
    # Length check
    if len(title) > 100:
        return False, "Title exceeds maximum length of 100 characters"
    
    # Block script injection
    if re.search(r"[<>\"'%;()&+]", title):
        return False, "Title contains invalid characters"
    
    return True, "Valid"

def lambda_handler(event, context):
    logger.info(f"Received event: {json.dumps(event)}")
    
    method = event.get("httpMethod", "GET")
    path   = event.get("path", "/")

    # ─── GET /todos ─────────────────────────────────────────────
    if method == "GET" and path == "/todos":
        logger.info("Fetching all todos")
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"todos": list(todo_store.values())})
        }

    # ─── POST /todos ─────────────────────────────────────────────
    elif method == "POST" and path == "/todos":
        try:
            body = json.loads(event.get("body") or "{}")
        except json.JSONDecodeError:
            logger.warning("Invalid JSON received")
            return {"statusCode": 400, "body": json.dumps({"error": "Invalid JSON"})}

        valid, message = validate_input(body)
        if not valid:
            logger.warning(f"Input validation failed: {message}")
            return {"statusCode": 400, "body": json.dumps({"error": message})}

        import uuid
        todo_id = str(uuid.uuid4())[:8]
        todo_store[todo_id] = {"id": todo_id, "title": body["title"].strip(), "done": False}

        logger.info(f"Created todo: {todo_id}")
        return {
            "statusCode": 201,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"message": "Todo created", "todo": todo_store[todo_id]})
        }

    # ─── Unknown route ───────────────────────────────────────────
    else:
        logger.warning(f"Unknown route: {method} {path}")
        return {"statusCode": 404, "body": json.dumps({"error": "Route not found"})}
```

Click **Deploy** to save.

📸 *Screenshot: Lambda code editor showing the function with input validation*

---

### STEP 4 — Test the Lambda Function Directly

Before connecting API Gateway, test the function:

1. Click the **Test** tab in Lambda
2. Create a new test event named `TestPost` with this body:

```json
{
  "httpMethod": "POST",
  "path": "/todos",
  "body": "{\"title\": \"Complete cloud security assignment\"}"
}
```

3. Click **Test** — you should see `Status: 201`
4. Now test input validation with a malicious input:

```json
{
  "httpMethod": "POST",
  "path": "/todos",
  "body": "{\"title\": \"<script>alert('xss')</script>\"}"
}
```

5. You should see `Status: 400` with `"error": "Title contains invalid characters"`

📸 *Screenshot: Successful 201 response and the 400 blocked malicious input*

---

### STEP 5 — Create API Gateway

1. Go to **API Gateway → Create API**
2. Choose **REST API** → click **Build**
3. Settings:
   - **API name:** `SecureServerlessAPI`
   - **Endpoint type:** Regional
4. Click **Create API**

#### Create the `/todos` Resource:

1. Click **Actions → Create Resource**
2. **Resource name:** `todos`
3. **Resource path:** `/todos`
4. Click **Create Resource**

#### Create GET method:

1. Select `/todos` → **Actions → Create Method → GET**
2. **Integration type:** Lambda Function
3. Enable **Use Lambda Proxy integration**
4. **Lambda function:** `SecureToDoAPI`
5. Click **Save** → **OK** to grant permission

#### Create POST method:

1. Select `/todos` → **Actions → Create Method → POST**
2. Same settings as GET — integration type: Lambda, proxy integration, `SecureToDoAPI`
3. Click **Save** → **OK**

📸 *Screenshot: API Gateway resource tree showing GET and POST under /todos*

---

### STEP 6 — Deploy the API

1. Click **Actions → Deploy API**
2. **Deployment stage:** [New Stage]
3. **Stage name:** `prod`
4. Click **Deploy**
5. Copy the **Invoke URL** shown — it looks like:
   ```
   https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod
   ```

📸 *Screenshot: Stage editor showing the Invoke URL*

---

### STEP 7 — Test the Live API via HTTP

Open **CloudShell** (top bar in AWS Console) or use your terminal.

#### Test GET:
```bash
curl https://YOUR_INVOKE_URL/todos
```
Expected response:
```json
{"todos": []}
```

#### Test POST (valid input):
```bash
curl -X POST https://YOUR_INVOKE_URL/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Secure serverless app"}'
```
Expected: `201 Created` with the new todo item.

#### Test POST (malicious input — should be blocked):
```bash
curl -X POST https://YOUR_INVOKE_URL/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "<script>xss</script>"}'
```
Expected: `400 Bad Request` — input validation working!

#### Test POST (empty title — should be blocked):
```bash
curl -X POST https://YOUR_INVOKE_URL/todos \
  -H "Content-Type: application/json" \
  -d '{"title": ""}'
```
Expected: `400 Bad Request`

📸 *Screenshot: Terminal showing all 4 curl commands and responses*

---

### STEP 8 — Enable and View CloudWatch Logs

Lambda automatically sends logs to CloudWatch if the IAM role has `AWSLambdaBasicExecutionRole`. Let's verify.

1. Go to **CloudWatch → Log groups**
2. Find `/aws/lambda/SecureToDoAPI`
3. Click the most recent **log stream**
4. You will see entries like:
   - `Received event: {...}`
   - `Created todo: abc123`
   - `Input validation failed: Title contains invalid characters`

📸 *Screenshot: CloudWatch log stream showing INFO and WARNING log entries*

---

### STEP 9 — Analyze Logs for Suspicious Activity

In CloudWatch, use **Log Insights** to query logs:

1. Go to **CloudWatch → Log Insights**
2. Select log group: `/aws/lambda/SecureToDoAPI`
3. Run this query to find all blocked/suspicious requests:

```sql
fields @timestamp, @message
| filter @message like /validation failed/
| sort @timestamp desc
| limit 20
```

4. Run this query to see all API calls:

```sql
fields @timestamp, @message
| filter @message like /Received event/
| sort @timestamp desc
| limit 20
```

📸 *Screenshot: CloudWatch Log Insights results showing flagged suspicious requests*

---

### STEP 10 — Analyze Serverless Security Risks

Document the following risks in your assignment report:

| Risk | Description | Mitigation Implemented |
|---|---|---|
| **Injection attacks** | Malicious input via API body | Input validation with regex |
| **Over-privileged function** | Lambda with admin rights | Least-privilege IAM role |
| **Missing logging** | No audit trail | CloudWatch logging enabled |
| **Insecure direct object reference** | Exposing raw IDs | UUIDs instead of sequential IDs |
| **No rate limiting** | DDoS on Lambda | API Gateway throttling (optional) |
| **Unencrypted transit** | HTTP exposure | API Gateway enforces HTTPS |

---

## Assignment Deliverables Checklist

- [ ] IAM execution role created with least privilege
- [ ] Lambda function deployed with input validation
- [ ] API Gateway configured with GET and POST endpoints
- [ ] Live API tested via curl (valid + invalid inputs)
- [ ] CloudWatch logs showing INFO and WARNING entries
- [ ] Log Insights query showing suspicious activity detection
- [ ] Risk analysis table in report
- [ ] Screenshots of every step above
- [ ] (Optional) Demo video walkthrough

---

## Bonus (Extra Credit)

### Add API Gateway Throttling (Rate Limiting):
1. In API Gateway → Stage → **Default Route Settings**
2. Set **Rate:** `100` requests/second, **Burst:** `50`
3. Screenshot the throttling settings

### Add a Usage Plan + API Key:
1. API Gateway → **Usage Plans → Create**
2. Attach to your `prod` stage
3. Generate an API Key and test with `-H "x-api-key: YOUR_KEY"`

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `{"message": "Internal server error"}` | Check CloudWatch logs for Python errors |
| `curl: (6) Could not resolve host` | Double check the Invoke URL is copied correctly |
| 403 on API Gateway | Re-deploy the API after adding methods |
| Lambda timeout | Increase timeout in Lambda → Configuration → General |
| No logs in CloudWatch | Verify the IAM role has `AWSLambdaBasicExecutionRole` |

---

*Prepared for ZG504 – Security Fundamentals for Cloud | BITS Pilani*
