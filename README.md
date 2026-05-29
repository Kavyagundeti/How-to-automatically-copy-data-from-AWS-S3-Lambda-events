# How-to-automatically-copy-data-from-AWS-S3-Lambda-events
# AWS S3 Auto File Mover with Lambda 🚀

> Automatically move files from one S3 bucket to another using Lambda triggers — no manual intervention required.

---

## 📌 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Services Used](#services-used)
- [Key Concepts](#key-concepts)
- [Step 1 — Create Two S3 Buckets](#step-1--create-two-s3-buckets)
- [Step 2 — Create IAM Policy](#step-2--create-iam-policy)
- [Step 3 — Create IAM Role](#step-3--create-iam-role)
- [Step 4 — Create Lambda Function](#step-4--create-lambda-function)
- [Step 5 — Add Bucket Policies](#step-5--add-bucket-policies)
- [Step 6 — Test Lambda Manually](#step-6--test-lambda-manually)
- [Step 7 — Add S3 Trigger](#step-7--add-s3-trigger)
- [Step 8 — Live End-to-End Test](#step-8--live-end-to-end-test)
- [IAM Permissions Explained](#iam-permissions-explained)
- [Common Errors & Fixes](#common-errors--fixes)
- [Interview Questions & Answers](#interview-questions--answers)
- [Key Rules to Remember](#key-rules-to-remember)

---

## Architecture Overview

```
User Uploads File
       ↓
Source S3 Bucket (images/ folder)
       ↓
S3 PUT Event Trigger (automatic)
       ↓
Lambda Function (Python)
       ↓
Copy file → Destination Bucket
       ↓
Delete file ← Source Bucket
       ↓
File available in Destination Bucket ✅
```

---

## Services Used

| Service | Purpose |
|---------|---------|
| **S3** | Store source and destination files |
| **Lambda** | Copy and delete files automatically |
| **IAM Policy** | Define minimum permissions |
| **IAM Role** | Assign permissions to Lambda |
| **CloudWatch** | View Lambda logs and errors |
| **S3 Event Notification** | Trigger Lambda on file upload |

---

## Key Concepts

### What is S3?
- **Simple Storage Service** — object-based storage in AWS
- Stores files as objects inside **buckets**
- Each object has a **key** (file path), **value** (data), **metadata**
- Highly durable — **99.999999999% (11 9's)**

### What is Lambda?
- **Serverless compute service** — runs code without managing servers
- Charges only for **execution time**
- Supports Python, Node.js, Java, Go etc.
- Max execution time = **15 minutes**
- Default timeout = **3 seconds**

### What is an S3 Trigger?
- An **event notification** from S3 to Lambda
- Fires automatically when a file is uploaded (**PUT event**)
- Can filter by **prefix** (folder) and **suffix** (file type)
- Zero manual intervention needed

### What is IAM?
- **Identity and Access Management** — controls who can do what in AWS
- **Policy** = list of permissions (JSON format)
- **Role** = identity assigned to a service like Lambda
- Always follow **Least Privilege Principle**

---

## Step 1 — Create Two S3 Buckets

1. Go to **S3 → Create bucket**
2. Enter a unique bucket name
3. Select your **AWS Region** (e.g., `ap-south-1`)
4. Turn **Block all public access → OFF**
5. Click **Create bucket**
6. Repeat for the second bucket

| Bucket | Purpose |
|--------|---------|
| `my-sourcebucket-197` | Upload files here |
| `my-destinationbucket-197` | Files arrive here automatically |

> ⚠️ Both buckets must be in the **same AWS region**

---

## Step 2 — Create IAM Policy

1. Go to **IAM → Policies → Create policy**
2. Click **JSON tab**
3. Paste the following JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetAndDelete",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-sourcebucket-197/images/*"
    },
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-sourcebucket-197"
    },
    {
      "Sid": "AllowPutDestination",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-destinationbucket-197/images/*"
    }
  ]
}
```

4. Click **Next**
5. Name it `S3MoveFilesPolicy`
6. Click **Create policy**

> ⚠️ Replace `my-sourcebucket-197` and `my-destinationbucket-197` with your actual bucket names

---

## Step 3 — Create IAM Role

1. Go to **IAM → Roles → Create role**
2. Select **AWS Service → Lambda → Next**
3. Search and attach these two policies:
   - ✅ `S3MoveFilesPolicy` *(custom policy created above)*
   - ✅ `AWSLambdaBasicExecutionRole` *(for CloudWatch logs)*
4. Click **Next**
5. Name it `s3-role`
6. Click **Create role**

---

## Step 4 — Create Lambda Function

1. Go to **Lambda → Create function**
2. Select **Author from scratch**
3. Fill in the details:

| Field | Value |
|-------|-------|
| Function name | `S3FileMover` |
| Runtime | Python 3.9 |
| Execution role | Use existing role → `s3-role` |

4. Click **Create function**
5. Paste the following code in the editor:

```python
import boto3

s3 = boto3.resource('s3')

def lambda_handler(event, context):
    source_bucket = s3.Bucket('my-sourcebucket-197')
    dest_bucket   = s3.Bucket('my-destinationbucket-197')

    print(f"Source: {source_bucket.name}")
    print(f"Destination: {dest_bucket.name}")

    for obj in source_bucket.objects.filter(Prefix='images/', Delimiter='/'):
        dest_key = obj.key
        print(f"Copying: {dest_key}")

        # Copy file to destination bucket
        s3.Object(dest_bucket.name, dest_key).copy_from(
            CopySource={'Bucket': obj.bucket_name, 'Key': obj.key}
        )
        print(f"Copied successfully: {dest_key}")

        # Delete original from source bucket
        s3.Object(source_bucket.name, obj.key).delete()
        print(f"Deleted from source: {dest_key}")
```

6. Click **Deploy**

> 💡 Change `Prefix='images/'` to `Prefix=''` to copy ALL files from the entire bucket

---

## Step 5 — Add Bucket Policies

### Source Bucket Policy

1. Go to **S3 → `my-sourcebucket-197` → Permissions tab**
2. Scroll to **Bucket policy → Edit**
3. Paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::my-sourcebucket-197/*"]
        }
    ]
}
```

4. Click **Save changes**

### Destination Bucket Policy

1. Go to **S3 → `my-destinationbucket-197` → Permissions tab**
2. Scroll to **Bucket policy → Edit**
3. Paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::my-destinationbucket-197/*"]
        }
    ]
}
```

4. Click **Save changes**

---

## Step 6 — Test Lambda Manually

1. Go to **Lambda → `S3FileMover`**
2. Click **Test → Create new event**
3. Name it `testevent`
4. Paste this JSON:

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "my-sourcebucket-197"
        },
        "object": {
          "key": "images/testfile.jpg"
        }
      }
    }
  ]
}
```

5. Click **Save → Test**
6. Check **Execution results** — should show ✅ Success
7. Verify file moved in S3 console

---

## Step 7 — Add S3 Trigger

1. Go to **Lambda → `S3FileMover` → Add trigger**
2. Fill in exactly:

| Setting | Value |
|---------|-------|
| Source | S3 |
| Bucket | `my-sourcebucket-197` |
| Event type | `PUT` |
| Prefix | `images/` |
| Suffix | *(leave empty)* |

3. Check the **acknowledgement checkbox**
4. Click **Add**

### Verify Trigger is Active

1. Go to **S3 → `my-sourcebucket-197` → Properties tab**
2. Scroll to **Event notifications**
3. Should show:

| Event | Prefix | Destination |
|-------|--------|-------------|
| PUT | `images/` | Lambda: S3FileMover |

---

## Step 8 — Live End-to-End Test

### Via AWS Console:
1. Go to **S3 → `my-sourcebucket-197` → `images/` folder**
2. Click **Upload → Add files**
3. Select any image file
4. Click **Upload**
5. Wait **5–10 seconds**

### Via AWS CLI:
```bash
aws s3 cp ./yourimage.jpg s3://my-sourcebucket-197/images/
```

### Expected Results:
- ✅ File appears in `my-destinationbucket-197/images/`
- ✅ File deleted from `my-sourcebucket-197/images/`
- ✅ Object URL opens image in browser

### View File in Browser:
```
https://my-destinationbucket-197.s3.amazonaws.com/images/yourfile.jpg
```

### Check CloudWatch Logs:
1. Go to **Lambda → Monitor tab → View CloudWatch logs**
2. Open the latest log stream
3. You should see:
```
Source: my-sourcebucket-197
Destination: my-destinationbucket-197
Copying: images/yourfile.jpg
Copied successfully: images/yourfile.jpg
Deleted from source: images/yourfile.jpg
```

---

## IAM Permissions Explained

| Permission | Applied On | Why Needed |
|------------|-----------|------------|
| `s3:GetObject` | Source `/images/*` | Read the file to copy it |
| `s3:DeleteObject` | Source `/images/*` | Delete file after copying |
| `s3:ListBucket` | Source bucket (root only) | Scan and list folder contents |
| `s3:PutObject` | Destination `/images/*` | Write the copied file |

> ⚠️ `s3:ListBucket` must point to the **bucket ARN** (no path).
> `s3:GetObject` and `s3:DeleteObject` must point to the **object ARN** (`/images/*`).

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDenied ListObjects` | Missing `s3:ListBucket` in policy | Fix IAM policy — Resource must be bucket ARN only, no `/images/*` |
| `AccessDenied` on image URL | Bucket not public | Disable Block public access + add bucket policy |
| `Policy has invalid resource` | Wrong ARN format | Must end with `/*` — no region or account ID in S3 ARN |
| Lambda not triggering | Uploaded to wrong folder | File must be uploaded inside `images/` folder exactly |
| Trigger not firing | Trigger not attached properly | Re-add trigger in Lambda → Add trigger |
| `NoSuchBucket` error | Bucket name typo in Lambda code | Check bucket names match exactly in Python code |

---

## Interview Questions & Answers

**Q: What is the difference between IAM Role and IAM Policy?**
> Policy defines **what actions** are allowed or denied. Role is an **identity** that carries those policies and is assigned to AWS services like Lambda.

**Q: Why use Lambda instead of EC2 for this task?**
> Lambda is **serverless** — no server management needed, auto-scales automatically, and you only pay per execution. EC2 runs 24/7 and costs more for simple event-driven tasks.

**Q: What is the Least Privilege Principle?**
> Always grant only the **minimum permissions** required to perform a specific task — nothing more, nothing less. This reduces security risks.

**Q: Why is `s3:ListBucket` on bucket ARN and not on object ARN?**
> `ListBucket` is a **bucket-level** operation. It needs access to the bucket itself. Object-level operations like `GetObject` use the object ARN with `/images/*`.

**Q: What happens if you remove the Prefix from the trigger?**
> Lambda will trigger on **every upload** anywhere in the bucket — not just the `images/` folder. This can cause unintended executions.

**Q: What is a serverless architecture?**
> No servers to provision or manage. AWS handles all infrastructure automatically. You only write and deploy code. Lambda, S3, and DynamoDB are serverless services.

**Q: How does S3 trigger Lambda?**
> S3 sends an **event notification** to Lambda when a PUT event occurs. Lambda receives an event JSON object containing the bucket name and object key automatically.

**Q: What is boto3?**
> The official AWS **SDK for Python**. Used to interact with AWS services like S3, EC2, DynamoDB programmatically inside Lambda functions.

**Q: What is the maximum execution time for Lambda?**
> **15 minutes** (900 seconds). Default timeout is 3 seconds.

**Q: What event type should be used to trigger Lambda on file upload?**
> **PUT** event type — it fires when a new object is created or uploaded to S3.

---

## Key Rules to Remember

| Rule | Detail |
|------|--------|
| Upload folder | Always upload to `images/` folder in source bucket |
| S3 ARN format | `arn:aws:s3:::bucketname` — **no region, no account ID** |
| `ListBucket` ARN | Bucket level only — **no** `/images/*` at the end |
| `GetObject` ARN | Object level — `arn:aws:s3:::bucket/images/*` |
| Trigger prefix | Case sensitive — `images/` ≠ `Images/` |
| Lambda role | Needs both `S3MoveFilesPolicy` + `AWSLambdaBasicExecutionRole` |
| Public access | Block public access OFF + bucket policy both required |
| Trigger event type | Must be **PUT** — not GET or DELETE |
| Bucket names in code | Must exactly match actual S3 bucket names |
| Same region | Both buckets must be in the same AWS region |

---

## Trigger Flow Summary

```
0 sec  → You upload file to images/ folder
1 sec  → S3 detects PUT event automatically
2 sec  → Lambda function starts
3-5 sec → File copied to destination bucket
5-6 sec → File deleted from source bucket
Done!  → File available in destination ✅
```

> You only upload the file — **everything else is fully automatic** 🎯

---

*Built with AWS S3 + Lambda + IAM + CloudWatch*
