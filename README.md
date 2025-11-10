# AWS Lambda + DynamoDB Workshop

A hands-on guide to deploying **FastAPI** with **AWS Lambda** and **DynamoDB** using the **Serverless Framework**.  
Learn the principles of **serverless architecture** and **clean architecture layering** while building a fully functional API.

---

## 🎯 Clean Architecture Layers

The project structure enforces a clear separation of concerns:

| Layer | Responsibility | Components |
| :--- | :--- | :--- |
| **Controllers** | **Interface/HTTP** - Handles incoming HTTP requests, validates input, and returns HTTP responses. | `controllers/` |
| **Use Cases** | **Business Logic** - Contains the core application rules and orchestrates data flow to and from the Repositories. | `usecases/` |
| **Repositories** | **Data Access** - Abstracts all interactions with the database (DynamoDB). | `repositories/`, `db/` |
| **Models** | **Data Structure** - Defines data shapes for request/response and database entities (using Pydantic). | `models/` |

**Why This Matters**: This separation ensures that the business logic (Use Cases) is isolated, making the application easier to test, maintain, and adapt to changes in the database or API structure.

## 🧩 Prerequisites

Before you begin, make sure you have the following installed:

- ✅ AWS Account with CLI configured (`aws configure`)
- 🐍 Python 3.11+
- 🧰 Node.js 18+ (for Serverless Framework)

### Quick Setup


# Install Serverless Framework
npm install -g serverless

# Install AWS CLI
pip install awscli

# Verify installation
serverless --version
aws --version


## Project Structure

```bash
apps/piowise-be/
├── controllers/        # API endpoints
├── repositories/       # DynamoDB operations
├── usecases/           # Business logic
├── models/             # Data models
├── db/                 # Database config
├── app.py              # FastAPI app entry point
├── serverless.yml      # Deployment configuration
├── requirements.txt    # Python dependencies
└── README.md
```
## Step 1: Setup Local Environment

```bash
cd apps/piowise-be
python -m venv venv

source venv/bin/activate  # macOS/Linux
or 
venv\\Scripts\\activate  # Windows

pip install -r requirements.txt
```

### requirements.txt

```yaml
fastapi==0.104.1
uvicorn==0.24.0
boto3==1.28.85
mangum==0.17.0
pydantic==2.5.0
python-dotenv==1.0.0
```

## Step 2: Create DynamoDB Table

Create a table in AWS:

```bash
aws dynamodb create-table \
  --table-name items \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-1
```

## Step 3: Build Your FastAPI App

### 3.1 Data Model

**models/item.py**:

```python
from pydantic import BaseModel
from typing import Optional

class Item(BaseModel):
    id: str
    name: str
    description: Optional[str] = None
    price: float
```

### 3.2 Database Configuration

**db/dynamodb.py**:

```python
import boto3
import os

dynamodb = boto3.resource(
    'dynamodb',
    region_name=os.getenv('AWS_REGION', 'ap-southeast-1')
)

def get_table(table_name: str):
    return dynamodb.Table(table_name)
```

### 3.3 Repository Layer

**repositories/item_repository.py**:

```python
from db.dynamodb import get_table
from models.item import Item

class ItemRepository:
    def __init__(self):
        self.table = get_table('items')
    
    def create(self, item: Item):
        self.table.put_item(Item=item.dict())
        return item
    
    def get(self, item_id: str):
        response = self.table.get_item(Key={'id': item_id})
        return response.get('Item')
    
    def list_all(self):
        response = self.table.scan()
        return response.get('Items', [])
    
    def delete(self, item_id: str):
        self.table.delete_item(Key={'id': item_id})
```

### 3.4 Use Case Layer

**usecases/item_usecase.py**:

```python
from repositories.item_repository import ItemRepository
from models.item import Item

class ItemUseCase:
    def __init__(self):
        self.repo = ItemRepository()
    
    def create_item(self, item: Item):
        return self.repo.create(item)
    
    def get_item(self, item_id: str):
        return self.repo.get(item_id)
    
    def list_items(self):
        return self.repo.list_all()
    
    def delete_item(self, item_id: str):
        self.repo.delete(item_id)
```

### 3.5 Controller Layer

**controllers/item_controller.py**:

```python
from fastapi import APIRouter, HTTPException
from models.item import Item
from usecases.item_usecase import ItemUseCase

router = APIRouter()
usecase = ItemUseCase()

@router.post("/")
def create_item(item: Item):
    return usecase.create_item(item)

@router.get("/{item_id}")
def get_item(item_id: str):
    item = usecase.get_item(item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    return item

@router.get("/")
def list_items():
    return usecase.list_items()

@router.delete("/{item_id}")
def delete_item(item_id: str):
    usecase.delete_item(item_id)
    return {"message": "Item deleted"}
```

### 3.6 Main FastAPI App

**app.py**:

```python
from fastapi import FastAPI
from mangum import Mangum
from controllers.item_controller import router as item_router

app = FastAPI(title="Lambda DynamoDB API")
app.include_router(item_router, prefix="/api/items", tags=["items"])

@app.get("/health")
def health():
    return {"status": "ok"}

# Lambda handler
handler = Mangum(app)
```

## Step 4: Test Locally with Uvicorn

```bash
# Run the app
uvicorn app:app --reload

# In another terminal, test endpoints
curl -X POST http://localhost:8000/api/items \
  -H "Content-Type: application/json" \
  -d '{"id":"1","name":"Widget","price":9.99}'

curl http://localhost:8000/api/items/1

curl http://localhost:8000/api/items

curl -X DELETE http://localhost:8000/api/items/1
```

## Step 5: Deploy to AWS Lambda

### 5.1 Serverless Configuration

**serverless.yml**:

```yaml
service: piowise-api

frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.11
  region: ap-southeast-1
  environment:
    AWS_REGION: ap-southeast-1
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:DeleteItem
            - dynamodb:Scan
          Resource: "arn:aws:dynamodb:ap-southeast-1:*:table/items"

functions:
  api:
    handler: app.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
          cors: true
      - http:
          path: /
          method: ANY
          cors: true

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true
```

### 5.2 Deploy Steps

```bash
# Install Serverless plugin
npm install --save-dev serverless-python-requirements

# Deploy to AWS
serverless deploy

# View logs
serverless logs -f api --tail

# Get API endpoint from deployment output
```

## Step 6: Test Deployed API

After deployment, use the API endpoint provided:

```bash
API_URL="https://your-api-id.execute-api.ap-southeast-1.amazonaws.com/dev"

# Create item
curl -X POST $API_URL/api/items \
  -H "Content-Type: application/json" \
  -d '{"id":"1","name":"Widget","price":9.99}'

# Get item
curl $API_URL/api/items/1

# List all items
curl $API_URL/api/items

# Health check
curl $API_URL/health
```

## Step 7: Cleanup

```bash
# Remove Lambda and API Gateway
serverless remove

# Delete DynamoDB table
aws dynamodb delete-table --table-name items --region ap-southeast-1
```

## Architecture Overview

**Clean Architecture Layers**:

1. **Controllers** - HTTP endpoints, request/response handling
2. **Use Cases** - Business logic and orchestration
3. **Repositories** - Data access and DynamoDB operations
4. **Models** - Data validation with Pydantic
5. **DB** - Database configuration and connection

**Why This Matters**:
- Easy to test each layer independently
- Changes to DynamoDB don't affect controllers
- Business logic is separate from HTTP concerns
- Reusable across different interfaces (CLI, events, etc.)

## Key Concepts

**AWS Lambda**: Serverless compute - pay only for execution time, auto-scales, event-driven.

**DynamoDB**: NoSQL database - fully managed, millisecond latency, automatic scaling.

**Mangum**: ASGI adapter that converts FastAPI to Lambda-compatible handler.

**Serverless Framework**: Infrastructure-as-code tool for deploying serverless applications.

## Resources

- [AWS Lambda Docs](https://docs.aws.amazon.com/lambda/)
- [DynamoDB Docs](https://docs.aws.amazon.com/dynamodb/)
- [FastAPI Docs](https://fastapi.tiangolo.com/)
- [Serverless Framework](https://www.serverless.com/)
- [Mangum](https://mangum.io/)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Lambda won't deploy | Check IAM permissions: \`aws sts get-caller-identity\` |
| DynamoDB connection fails | Verify table exists: \`aws dynamodb list-tables\` |
| Local testing fails | Ensure Uvicorn is running on port 8000 |
| Permission denied on deploy | Run \`aws configure\` and verify AWS credentials |
