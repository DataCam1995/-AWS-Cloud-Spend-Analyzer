# AWS Cloud Spend Analyzer

This project is a serverless cost monitoring solution that automatically alerts users when their AWS spending exceeds a predefined threshold. It leverages Amazon S3, AWS Glue, Amazon Athena, AWS Lambda, and Amazon SNS to ingest, transform, and analyze AWS Cost and Usage Reports (CUR) in near real-time.

---

## 📦 Architecture Overview

**Components Used:**
- **S3**: Stores the AWS CUR reports (`cost-report-bucket25`)
- **AWS Glue**: Crawls CUR data and catalogs it in Athena
- **Athena**: Queries daily/monthly spend from the CUR
- **Lambda**: Runs an Athena query and triggers an SNS alert if spend exceeds threshold
- **SNS**: Sends email notifications to subscribed users

---

## 🛠️ Setup Instructions

### 1. Enable AWS Cost and Usage Report
- Go to the AWS Billing Console → Cost & Usage Reports
- Set report to save to `s3://cost-report-bucket25/`
- Ensure the report includes resource IDs and is delivered in Parquet format

### 2. Set Up Glue Crawler
- Source: S3 path to the CUR bucket
- Database: `cost_usage_db`
- Run crawler to create the CUR table

### 3. Query Cost in Athena
Example query:
```sql
SELECT
  SUM(line_item_blended_cost) AS total_cost
FROM cost_usage_db.<cur_table_name>
WHERE year = '2025' AND month = '07';
```

### 4. Deploy Lambda Function
- Language: Python 3.12
- Create a Lambda function (e.g., `aws-cost-alert-lambda`)
- Paste the code from `lambda_function.py`
- Add the following **Environment Variables**:
  - `ATHENA_DB`: `cost_usage_db`
  - `ATHENA_TABLE`: `<your_table_name>`
  - `SNS_TOPIC_ARN`: `<your_SNS_topic_ARN>`
  - `COST_THRESHOLD`: `50`

### 5. Permissions
Lambda IAM Role must include:
- `AthenaFullAccess`
- `SNSFullAccess`
- `S3ReadOnlyAccess`
- `GlueReadOnlyAccess`

---

## ✅ Example Lambda Code

```python
import boto3, os, time

athena = boto3.client("athena")
sns = boto3.client("sns")

DATABASE = os.environ["ATHENA_DB"]
TABLE = os.environ["ATHENA_TABLE"]
COST_THRESHOLD = float(os.environ["COST_THRESHOLD"])
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]

def lambda_handler(event, context):
    query = f"SELECT SUM(line_item_blended_cost) FROM {DATABASE}.{TABLE};"
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": DATABASE},
        ResultConfiguration={"OutputLocation": f"s3://{TABLE}-query-results/"}
    )
    query_execution_id = response["QueryExecutionId"]

    while True:
        status = athena.get_query_execution(QueryExecutionId=query_execution_id)["QueryExecution"]["Status"]["State"]
        if status in ["SUCCEEDED", "FAILED", "CANCELLED"]:
            break
        time.sleep(2)

    if status != "SUCCEEDED":
        raise Exception(f"Athena query failed: {status}")

    result = athena.get_query_results(QueryExecutionId=query_execution_id)
    total_cost = float(result["ResultSet"]["Rows"][1]["Data"][0]["VarCharValue"])

    if total_cost >= COST_THRESHOLD:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="AWS Cost Alert",
            Message=f"🚨 Total AWS spend this month has exceeded ${COST_THRESHOLD:.2f}. Current cost: ${total_cost:.2f}"
        )

    return {"statusCode": 200, "body": f"Checked cost: ${total_cost:.2f}"}
```

---

## 📊 Quicksight Dashboard (Coming Soon)
You will visualize daily and monthly AWS spending trends with:
- Bar charts for service-level costs
- Line graphs showing daily accumulation
- Threshold markers to track spend goals

---

## 🙌 Future Improvements
- Add Slack/Teams alerts
- Daily scheduling with CloudWatch Events
- Real-time alerts via EventBridge and Lambda

---

## 🧠 Author
Built with 💻 by Cam Battle – Cost visibility = Cloud control.
