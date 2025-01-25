# Deploying a Serverless Web Application on AWS

## Overview
This guide walks you through deploying a serverless web application using AWS services like DynamoDB, Lambda, API Gateway, S3, and CloudFront. The application allows users to manage student data with features to save and view student information.

---

## Step 1: Create a DynamoDB Table

1. **Navigate to DynamoDB** in the AWS Management Console.
   - Select **Create Table**.
   - Table name: `StudentData`
   - Partition key: `StudentId`
   - Use default settings.
   - Click **Create Table**.

---

## Step 2: Create Lambda Functions

### 1. Create `GetStudent` Function

1. Open **AWS Lambda**.
   - Select **Create Function**.
   - Function name: `GetStudent`
   - Runtime: `Python 3.12`.
   - Execution Role: Create a new role or use an existing one with DynamoDB permissions.
   - Click **Create Function**.
2. Replace the default code with the following:

```python
import json
import boto3

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb', region_name='us-east-2')
    table = dynamodb.Table('StudentData')
    response = table.scan()
    data = response['Items']

    while 'LastEvaluatedKey' in response:
        response = table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
        data.extend(response['Items'])

    return data
```

3. Deploy and test the function.

### 2. Create `InsertStudentData` Function

1. Open **AWS Lambda**.
   - Select **Create Function**.
   - Function name: `InsertStudentData`
   - Runtime: `Python 3.12`.
   - Execution Role: Use the same role as above.
   - Click **Create Function**.
2. Replace the default code with the following:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('StudentData')

def lambda_handler(event, context):
    student_id = event['studentid']
    name = event['name']
    student_class = event['class']
    age = event['age']

    response = table.put_item(
        Item={
            'studentid': student_id,
            'name': name,
            'class': student_class,
            'age': age
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Student data saved successfully!')
    }
```

3. Deploy and test the function with this event:

```json
{
  "studentid": "1",
  "name": "John Doe",
  "class": "10",
  "age": "15"
}
```

---

## Step 3: Set Up API Gateway

1. Open **Amazon API Gateway**.
   - Create a new **REST API**.
   - API Name: `StudentAPI`.
2. Create Methods:
   - **GET** Method:
     - Integration Type: **Lambda Function**.
     - Lambda Function: `GetStudent`.
   - **POST** Method:
     - Integration Type: **Lambda Function**.
     - Lambda Function: `InsertStudentData`.
3. Deploy the API:
   - Create a **New Stage**.
   - Stage Name: `prod`.
   - Deploy.
4. Enable **CORS** for both methods.

---

## Step 4: Configure S3 for Static Website Hosting

1. Open **S3**.
   - Create a new bucket (e.g., `myprojectbuc`).
2. Upload `index.html` and `scripts.js` to the bucket.
3. Enable **Static Website Hosting**:
   - Index document: `index.html`.
4. Update Bucket Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::myprojectbuc/*"
    }
  ]
}
```

---

## Step 5: Set Up CloudFront

1. Open **CloudFront**.
   - Create a new **Distribution**.
   - Origin Domain: S3 bucket.
   - Default Root Object: `index.html`.
2. Update S3 Bucket Policy to integrate with CloudFront.

---

## Frontend Code

### `scripts.js`

```javascript
var API_ENDPOINT = "<YOUR_API_ENDPOINT>";

document.getElementById("savestudent").onclick = function() {
    var inputData = {
        "studentid": $('#studentid').val(),
        "name": $('#name').val(),
        "class": $('#class').val(),
        "age": $('#age').val()
    };
    $.ajax({
        url: API_ENDPOINT,
        type: 'POST',
        data: JSON.stringify(inputData),
        contentType: 'application/json; charset=utf-8',
        success: function(response) {
            document.getElementById("studentSaved").innerHTML = "Student Data Saved!";
        },
        error: function() {
            alert("Error saving student data.");
        }
    });
};

document.getElementById("getstudents").onclick = function() {
    $.ajax({
        url: API_ENDPOINT,
        type: 'GET',
        contentType: 'application/json; charset=utf-8',
        success: function(response) {
            $('#studentTable tr').slice(1).remove();
            response.forEach(data => {
                $("#studentTable").append(`<tr>
                    <td>${data['studentid']}</td>
                    <td>${data['name']}</td>
                    <td>${data['class']}</td>
                    <td>${data['age']}</td>
                </tr>`);
            });
        },
        error: function() {
            alert("Error retrieving student data.");
        }
    });
};
```

### `index.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Student Data</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>Manage Student Data</h1>
        <input type="text" id="studentid" placeholder="Student ID">
        <input type="text" id="name" placeholder="Name">
        <input type="text" id="class" placeholder="Class">
        <input type="text" id="age" placeholder="Age">
        <button id="savestudent">Save Student</button>
        <button id="getstudents">Get Students</button>

        <table id="studentTable">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Class</th>
                    <th>Age</th>
                </tr>
            </thead>
            <tbody></tbody>
        </table>
    </div>
    <script src="scripts.js"></script>
</body>
</html>
```
