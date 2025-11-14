# Trigger-a-Lambda-function-by-uploading-an-object-to-Amazon-S3
# AWS Lambda & S3 Image Thumbnail Generator Lab

## Overview

In this lab, you'll build a simple serverless application on AWS: a Lambda function that’s triggered by S3 bucket events to create thumbnail images. This hands-on guide walks you through the entire workflow—setting up storage, coding the Lambda function, configuring triggers, and monitoring everything with CloudWatch.

---

## Lab Objectives

By completing this lab, you will:

- **Create an AWS Lambda function**
- **Configure an Amazon S3 bucket as a Lambda Event Source**
- **Trigger a Lambda function by uploading objects to Amazon S3**
- **Monitor Lambda functions using Amazon CloudWatch Logs**

---

## Icon Key

This guide uses the following icons for special notes:

- ⚠️ **Caution:** Important info to avoid issues or unexpected results.
- 💡 **Note:** Helpful tip or guidance.
- ✅ **Task complete:** Summary or endpoint of a step.
- 🔥 **Warning:** Critical info to prevent data loss, security issues, or system damage.

---

## Starting the Lab

1. **Start the Lab**
   - At the top of the lab page, click **Start Lab**.
   - ⚠️ **Caution:** Wait for all AWS resources to be provisioned before continuing.

2. **Open the Lab Console**
   - Click **Open Console**. This signs you in to the AWS Management Console automatically in a new browser tab.
   - 🔥 **Warning:** **Do not** change the AWS Region unless the instructions tell you to.

#### Common Sign-in Errors

- If clicking **Start Lab** does nothing, browser pop-up or script blockers might be to blame:
  - Add the lab site to your browser’s allow list or disable script blockers.
  - Refresh and try again.

---

## Lab Scenario and Application Flow

You’ll build a serverless image thumbnail service, as illustrated below:

### Application Flow

1. Upload an image to a **source S3 bucket**.
2. **Amazon S3** detects the upload (object-created event).
3. S3 invokes a **Lambda function** and passes the event data.
4. **Lambda** retrieves the image, creates a thumbnail, and writes it to a **target bucket**.

When finished, your AWS account will contain:

- Two S3 buckets: one for uploads, one for thumbnails
- A functional Lambda configured to resize images upon upload
- Associated IAM permissions, VPC settings, and CloudWatch logs

---

## Task 1: Create the Amazon S3 Buckets

You need two buckets: one for image uploads, another for thumbnails.

1. In the AWS Console, search for and select **S3**.
2. Click **Create bucket**.
3. Name your bucket `images-<randomnumber>` (e.g. `images-123456789`).
   - Bucket names must be globally unique.
   - 💡 **Note:** If the name is taken, click *Edit*, choose a new random number, and try again.
4. Save the bucket name. You’ll need it later.
5. Accept default options and create the bucket.
6. **Create the output bucket:**  
   - Name it the same as above, but add `-resized` (e.g. `images-123456789-resized`).
   - Your buckets should now look like:  
     - `images-123456789`  
     - `images-123456789-resized`

#### Upload the Sample Image

1. Download [HappyFace.jpg](#) (use "Save link as…").
2. In the AWS Console, open your first (input) bucket.
3. Click **Upload** → **Add files** and select `HappyFace.jpg`.
4. Click **Upload**.

💡 **Note:** The Lambda function will refer to this file for its input event.

✅ **Task complete:** S3 buckets are ready and an image is uploaded.

---

## Task 2: Create the AWS Lambda Function

You’ll create a Lambda function that resizes images from S3.

1. Go to **Lambda** via the AWS Console search.
2. Click **Create function** > **Author from scratch**.
3. Enter:
   - **Function name:** `Create-Thumbnail`
   - **Runtime:** Python 3.12

### Configure the Execution Role

1. Expand **Change default execution role**.
2. Select **Use an existing role**.
3. Choose the role named `lambda-execution-role`.  
   💡 This gives Lambda read/write access to S3.

### Attach Networking (VPC, Subnet, Security Group)

1. Expand **Additional configurations**.
2. Select **VPC**.
3. For VPC, choose: **VPC with CIDR 10.0.0.0/16**
4. For Subnets: **10.0.1.0/24**
5. For Security groups: choose one with `LambdaSecurityGroup` in its name.

6. Click **Create function**.

💡 **Note:** It may take a few minutes for Lambda to provision resources. Wait for the “Successfully created the function” banner to appear.

---

### Connect Lambda to S3 Events

1. In your function's page, click **Add trigger**.
2. Select **S3** as the source.
3. For **Bucket**, select your `images-...` bucket (not the resized one).
4. Set **Event type** to *All object create events*.
5. Check **Recursive invocation**: “I acknowledge that...” (prevents infinite loops).
6. Click **Add**.

---

### Upload the Lambda Code

1. Go to the **Code** tab.
2. Select **Upload from** → **Amazon S3 location**.
3. Paste the provided Amazon S3 link URL (for `CreateThumbnail.zip`).
4. Click **Save**.

**Code included in the ZIP (for reference):**

```python
import boto3
import os
import sys
import uuid
from PIL import Image

s3_client = boto3.client('s3')

def resize_image(image_path, resized_path):
    with Image.open(image_path) as image:
        image.thumbnail((128, 128))
        image.save(resized_path)

def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        download_path = f'/tmp/{uuid.uuid4()}{key}'
        upload_path = f'/tmp/resized-{key}'

        s3_client.download_file(bucket, key, download_path)
        resize_image(download_path, upload_path)
        s3_client.upload_file(upload_path, f'{bucket}-resized', key)
```

- **Process:**
  - Receives event with bucket/key
  - Downloads image from S3
  - Resizes image (128x128) with Pillow (PIL)
  - Uploads output to the `-resized` bucket

---

### Set the Lambda Handler

1. In **Runtime settings**, click **Edit**.
2. Set **Handler** to: `CreateThumbnail.handler`.
3. Click **Save**.

---

### Update Description and Confirm Config

1. Go to the **Configuration** tab.
2. Select **General configuration** > **Edit**.
3. Add a **Description:** "Create a thumbnail-sized image."
4. Click **Save**.

💡 **Note:**  
- **Memory** sets CPU/Memory allocation.  
- **Timeout** controls max execution time.

✅ **Task complete:** Your Lambda function is now ready.

---

## Task 3: Test Your Lambda Function

You can simulate a real S3 event to test everything.

1. Go to the **Test** tab in Lambda.
2. Click **Create new event**.
3. Name the event: `Upload`.
4. Select template: **S3 Put**.
5. In the JSON:
   - Replace `example-bucket` (appears *twice*) with your input bucket name.
   - Replace `test%2Fkey` with `HappyFace.jpg`.
6. Click **Test**.

If all is well, you’ll see `Executing function: succeeded`.

- Click **Details** to view:
  - Execution time
  - Resource usage
  - Full log output

#### View the Resized Image

1. Go to S3, open your `-resized` bucket.
2. Find `HappyFace.jpg`, select and **Open** it.
   - Depending on browser, it may open or download.

The image should be a **thumbnail** version of your original.

💡 Try uploading your own images and see thumbnails appear in the resized bucket!

✅ **Task complete:** Lambda was triggered and the resized image created.

---

## Task 4: Monitor and Observe Logging

AWS Lambda provides monitoring tools via CloudWatch.

1. In AWS Console, go to **Lambda**.
2. Open your `Create-Thumbnail` function.
3. Click the **Monitor** tab.

### Metrics to Observe

- **Invocations:** Number of calls.
- **Duration:** Execution times.
- **Error count/success rate:** Error details.
- **Throttles:** When concurrency limits hit.
- **Async delivery failures:** Errors in notifications or dead-letter queues.
- **Iterator Age:** For stream events (not used here).
- **Total concurrent executions:** How many executions in parallel.

### View Logs in CloudWatch

1. Click **View CloudWatch logs**.
2. Select your function’s **Log Stream**.
3. Expand messages for details:
   - Request IDs
   - Duration
   - Memory use
   - All `print` and logging output
   - Error traces, if any

Logs help debug and improve serverless workflows.

✅ **Task complete:** You are monitoring Lambda S3 functions using CloudWatch.

---

## Knowledge Check

To confirm your understanding:
- **Why is S3 triggering Lambda useful?**
    - It automates image processing or other backend tasks when a new object is uploaded.
- **What libraries and permissions does your Lambda need?**
    - The code uses `boto3` (AWS SDK) and `Pillow` (image processing). The execution role needs S3 read/write permissions.
- **Where do you go to monitor logs and errors?**
    - AWS Lambda’s **Monitor** tab and **CloudWatch Logs**.

---

## Conclusion

Congratulations! You have:

- **Created an AWS Lambda function**
- **Set up Amazon S3 as an Event Source**
- **Triggered Lambda by uploading to S3**
- **Monitored Lambda execution with CloudWatch Logs**

This pattern is broadly useful for scalable, serverless workflows like:
- Automatic image conversion/thumbnailing
- Video/audio transcoding
- Metadata extraction
- Automated pipelines on new data

---

## End the Lab

To finish up:

1. Return to the **AWS Console**.
2. Click your username (**AWSLabsUser**) in the top-right.
3. Select **Sign out**.
4. Click **End Lab** and confirm.

---

## Additional Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [Monitoring AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics.html)
- [CloudWatch Logs Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/)
- [Pillow Python Imaging Library](https://pillow.readthedocs.io/en/stable/)

---
