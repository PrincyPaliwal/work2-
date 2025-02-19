# AI-Powered Data Quality & Anomaly Detection

## Overview
Ensuring high-quality data is crucial for accurate analytics and decision-making. This repository automates data cleansing, anomaly detection, and real-time alerting using **AWS Glue DataBrew, Databricks AutoML, and AWS CloudWatch**. It integrates seamlessly with AWS and Databricks to help teams monitor, govern, and enhance data quality efficiently.

## Features
- **Automated Data Cleaning**: Uses AWS Glue DataBrew for preprocessing.
- **AI-Powered Anomaly Detection**: Utilizes Databricks AutoML for real-time monitoring.
- **Real-Time Alerting**: AWS CloudWatch integration for instant notifications.
- **Cloud-Native Deployment**: Easily integrates with AWS Lambda and Databricks Jobs.

## Prerequisites
Ensure you have:
- **Databricks Workspace** (with appropriate AWS permissions)
- **AWS IAM Roles** (with access to DataBrew, CloudWatch, and SNS)
- **Databricks API Token** (for AutoML and job execution)
- **AWS CLI configured** (`aws configure`)

## Installation
Clone the repository:
```bash
git clone https://github.kadellabs.com/digiclave/databricks-accelerators.git
cd databricks-accelerators
```

## Code

### **Import Required Libraries**
```python
import boto3 
import requests 
import json 
import os 
import time 
import logging 

logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s") 
log = logging.getLogger(__name__) 

databrew = boto3.client("databrew") 
cloudwatch = boto3.client("cloudwatch") 
sns = boto3.client("sns")
```

### **Environment Variables**
```python
DATABRICKS_HOST  
DATABRICKS_TOKEN
DATASET_NAME  
PROJECT_NAME 
SNS_TOPIC_ARN
```

### **Create DataBrew Job**
```python
def create_databrew_job(): 
    try: 
        response = databrew.create_recipe( 
            Name="DataCleaningRecipe", 
            Steps=[ 
                { "Action": {"Operation": "REMOVE_NULLS"}, "ConditionExpressions": [{"Condition": "IS_NULL", "TargetColumn": "transaction_amount"}] }, 
                { "Action": {"Operation": "FILL_WITH_MEAN"}, "ConditionExpressions": [{"Condition": "IS_MISSING", "TargetColumn": "transaction_amount"}] } 
            ] 
        ) 
        log.info(f"✅ DataBrew Recipe Created: {response['Name']}") 
    except Exception as e:
        log.error(f"❌ Error creating DataBrew job: {e}")
```

### **Train AutoML Model in Databricks**
```python
def train_automl_model(): 
    headers = {"Authorization": f"Bearer {DATABRICKS_TOKEN}", "Content-Type": "application/json"} 
    payload = { 
        "name": "AnomalyDetectionModel", 
        "dataset_path": f"/mnt/{os.environ['DATABRICKS_MOUNT']}/cleaned_data", 
        "target_col": "transaction_amount", 
        "problem_type": "regression", 
        "timeout_minutes": 30 
    } 
    response = requests.post( 
        f"{DATABRICKS_HOST}/api/2.0/mlflow/experiments/create", 
        headers=headers, 
        json=payload 
    ) 
    if response.status_code == 200: 
        experiment_id = response.json()["experiment_id"] 
        log.info(f"✅ AutoML Model Training Started. Experiment ID: {experiment_id}") 
        return experiment_id 
    else: 
        log.error(f"❌ Error starting AutoML model training: {response.text}") 
        return None 
```

### **Wait for AutoML Model Completion**
```python
def wait_for_automl_completion(experiment_id): 
    headers = {"Authorization": f"Bearer {DATABRICKS_TOKEN}"} 
    while True: 
        response = requests.get( 
            f"{DATABRICKS_HOST}/api/2.0/mlflow/experiments/get?experiment_id={experiment_id}", 
            headers=headers 
        ) 
        status = response.json().get("lifecycle_stage") 
        if status == "active": 
            log.info("⚙️ AutoML Model is still training...") 
            time.sleep(30) 
        else: 
            log.info("✅ AutoML Model Training Completed!") 
            return 
```

### **Check for Anomalies**
```python
def check_anomalies(experiment_id): 
    headers = {"Authorization": f"Bearer {DATABRICKS_TOKEN}"} 
    response = requests.get( 
        f"{DATABRICKS_HOST}/api/2.0/mlflow/runs/search?experiment_id={experiment_id}", 
        headers=headers 
    ) 
    if response.status_code == 200: 
        runs = response.json()["runs"] 
        latest_run = runs[0] 
        metrics = latest_run["data"]["metrics"] 
        if metrics["rmse"] > 1000:  # Threshold for anomaly detection 
            log.warning("🚨 Anomaly Detected! Sending Alert...") 
            send_alert() 
        else: 
            log.info("✅ No anomalies detected.") 
    else: 
        log.error(f"❌ Error fetching model results: {response.text}") 
```

### **Contributions**
We welcome contributions! Submit a pull request or open an issue for feature requests or improvements.

### **License**
This project is licensed under the MIT License.

