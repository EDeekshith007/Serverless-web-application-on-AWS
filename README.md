# Deploying a Serverless Web Application on AWS

## Overview
This guide walks you through deploying a serverless web application using AWS services like DynamoDB, Lambda, API Gateway, S3, and CloudFront. The application allows users to manage student data with features to save and view student information.

## Architecture

![Image](https://github.com/user-attachments/assets/2443d7ee-4ea3-4ff2-8806-cb5964ed808d)

---

## Step 1: Create a DynamoDB Table

1. **Navigate to DynamoDB** in the AWS Management Console.
   
   ![Image](https://github.com/user-attachments/assets/a15e535d-b792-4d45-a749-e0d33d3f6f10)
   
   - Select **Create Table**.
   - Table name: `StudentData`
   - Partition key: `StudentId`
   - Use default settings.
   - Click **Create Table**.

      ![Image](https://github.com/user-attachments/assets/08891f94-3cd3-49a9-a6e6-0e8ee516efdb)
---

## Step 2: Create Lambda Functions

### 1. Create `GetStudent` Function

1. **Navigate to AWS Lambda** in the AWS Management Console.

        ![Image](https://github.com/user-attachments/assets/6b6d8ad8-8339-4f72-bd1b-46f61174cebd)
   
   - Select **Create Function**.
   - Function name: `GetStudent`
   - Runtime: `Python 3.12`.
   - Execution Role: Create a new IAM role or use an existing one with DynamoDB permissions.

           ![Image](https://github.com/user-attachments/assets/94620f40-b12c-4a91-9942-c4088ce53bb4)

              ![Image](https://github.com/user-attachments/assets/f4c4a199-a800-47a4-a50a-0745f4bc3e08)
     
   - Click **Create Function**.

     ![Image](https://github.com/user-attachments/assets/3775b4e4-0347-4043-98ef-0470e55da41b)
     
2. Write the following python:
   ![Image](https://github.com/user-attachments/assets/fd1d4eff-e704-48e2-a4fe-4a3d562b4941)
   

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

1. Open **AWS Lambda** Console again:
   - Select **Create Function**.
   - Function name: `InsertStudentData`
   - Runtime: `Python 3.12`.
   - Execution Role: Use the same role as above.
   - Click **Create Function**.

     ![Image](https://github.com/user-attachments/assets/7ae36bff-b5c5-4e44-b8d8-a0c6a4f41c97)
     
2. Write the following python:

   ![Image](https://github.com/user-attachments/assets/3c5cdaf5-1a85-4466-8d27-3a07622ca6eb)

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

   ![Image](https://github.com/user-attachments/assets/f8144dbd-9ee3-405b-91dd-b458037c76db)

   ![Image](https://github.com/user-attachments/assets/3aef8c61-6e9a-4aec-99be-43b06adadbe2)

4. Event JSON Code:

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

![Image](https://github.com/user-attachments/assets/382c8955-b441-4c47-84f9-16f7ed8c4450)

   - Create a new **REST API**.
   - API Name: `StudentAPI`.
     
     ![Image](https://github.com/user-attachments/assets/470ed516-6683-4eca-af88-16a6a71eae5b)
     
2. Create Methods:

     ![Image](https://github.com/user-attachments/assets/4cff5fb3-5de9-4216-8de6-d04a48169d0e)
   
   - **GET** Method:
     - Integration Type: **Lambda Function**.
     - Lambda Function: `GetStudent`.

         ![Image](https://github.com/user-attachments/assets/b8783996-4a74-4822-8c8a-83304280c5bd)
       
   - **POST** Method:
     - Integration Type: **Lambda Function**.
     - Lambda Function: `InsertStudentData`.
    
       ![Image](https://github.com/user-attachments/assets/4e552424-c0d5-4764-a4d6-e248643d94e2)
       
3. Deploy the API:
   - Create a **New Stage**.
   - Stage Name: `prod`.
   - Deploy.
4. Enable **CORS** for both methods.

   ![Image](https://github.com/user-attachments/assets/e10836ea-d35b-4fd9-a34f-4ae61ad58b11)

---

## Step 4: Configure S3 for Static Website Hosting

![Image](https://github.com/user-attachments/assets/548ff82a-228c-4d82-a557-b0eab60c6f82)

1. Open **S3**.
   - Create a new bucket (e.g., `myprojectbuc`).
     
     ![Image](https://github.com/user-attachments/assets/ef5a2c7b-2e62-4302-8a1b-17b0a78e450b)
     
2. Upload `index.html` and `scripts.js` to the bucket.

   ![Image](https://github.com/user-attachments/assets/2dfc3f63-b229-446c-8c63-ddcfa3247150)
   
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

![Image](https://github.com/user-attachments/assets/4868d1fe-ac95-4d59-b113-8961839e0390)

1. Open **CloudFront**.
   - Create a new **Distribution**.

    ![Image](https://github.com/user-attachments/assets/7b790f52-b8db-4e19-880e-c074f3b5795f)

    ![Image](https://github.com/user-attachments/assets/328f5d2a-f619-46b4-9146-9a5f5011b100)

2.Use your S3 bucket as the origin and create an Origin Access Control (OAC).
   - Origin Domain: S3 bucket.
   - Default Root Object: `index.html`.
3. Update your S3 bucket policy to block all public access and allow CloudFront access.
4. Copy the CloudFront distribution domain and test the application.

   ![Image](https://github.com/user-attachments/assets/1bfc1afd-7a46-4c80-97ac-9c103a5d2f36)
   ![Image](https://github.com/user-attachments/assets/6b195bfb-1c62-4375-8c8a-309ffc2f7eda)

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


![Image](https://github.com/user-attachments/assets/d7b8afa3-bd33-458f-80ef-7bd2c8471a5b)


## Conclusion

Successfully deployed a serverless web application on AWS.
