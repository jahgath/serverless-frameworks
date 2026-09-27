# AWS Serverless Patterns Workshop

This project walks through the AWS Serverless Patterns workshop, building a small food-ordering platform module by module to demonstrate the recurring architecture patterns behind serverless applications: synchronous request/response, idempotence, asynchronous event-driven processing, and long-polling status checks.

The project was created as part of an Enterprise Software Platforms assignment at San Jose State University.

Workshop: https://catalog.workshops.aws/serverless-patterns/en-US

## Architecture

Each module deploys as its own AWS SAM stack, built with the AWS SAM CLI and deployed to a single AWS account/region (`us-east-1`). Later modules depend on the Cognito user pool created in Module 2.

```text
                        AWS SAM CLI
                             |
                       sam build / deploy
                             |
                     AWS CloudFormation
                    /        |        \
                   /         |         \
              Module 2   Module 3    Module 4
              (Users)    (Orders)  (User Profile)
             Cognito +   API GW +  EventBridge +
             DynamoDB +  Lambda +  SQS + Lambda +
             API GW      DynamoDB  API GW + DynamoDB
                  \          |          /
                   \         |         /
                    Shared Cognito User Pool
```

## Technologies Used

- AWS SAM (Serverless Application Model) CLI
- AWS Lambda (Python 3.12)
- Amazon API Gateway
- Amazon DynamoDB
- Amazon Cognito
- Amazon EventBridge
- Amazon SQS
- AWS Lambda Powertools (structured logging, metrics, idempotency)
- Amazon CloudWatch (logs, metrics, X-Ray tracing)
- pytest (integration tests)

## Project Structure

```text
ws-serverless-patterns/
├── README.md
├── template.yaml         # Root stack wiring in the Users service
├── module1/               # Module 1 - Intro to Serverless
│   └── m1-add-sample-data.py
├── module2/               # Module 2 - Synchronous Invocation (Users service)
│   ├── template.yaml
│   ├── src/api/
│   └── tests/
├── users/                 # Base Users/Cognito stack (prerequisite for modules 3-4)
│   ├── template.yaml
│   ├── src/api/
│   └── tests/
├── orders/                 # Module 3 - Synchronous + Idempotence (Orders service)
│   ├── template.yaml
│   ├── src/api/order/
│   └── tests/integration/
└── userprofile/             # Module 4 - Asynchronous Invocation (Address + Favorites)
    ├── template.yaml
    ├── api.yaml
    ├── src/api/
    └── tests/integration/
```

## Module 2 — Synchronous Invocation

Deploys a `Users` microservice: a Cognito user pool, a DynamoDB table, and a Lambda-backed REST API behind API Gateway with a Lambda token authorizer. Every later module authenticates against this same user pool.

## Module 3 — Synchronous + Idempotence

Deploys an `Orders` microservice with Lambda functions to add, get, list, edit, and cancel orders, backed by DynamoDB and a shared Lambda layer (`pyutils`). The `AddOrder` operation is made idempotent with AWS Lambda Powertools, so retried requests (e.g. from a flaky client) return the original result instead of creating duplicate orders or failing. Structured logging and custom CloudWatch metrics (`SuccessfulOrder`, `OrderTotal`) are added via Powertools.

Run the integration tests with:

```bash
cd orders
source venv/bin/activate
export USERS_STACK_NAME=<users-stack-name>
export ORDERS_STACK_NAME=ws-serverless-patterns-orders
pytest tests/integration -v
```

## Module 4 — Asynchronous Invocation

Deploys a `UserProfile` service with two asynchronous patterns:

- **Address changes** (add/edit/delete) go through Amazon EventBridge: API Gateway publishes an event directly to an event bus, and per-operation Lambda functions react via EventBridge rules.
- **Favorites** (add/delete) go through an Amazon SQS queue: API Gateway sends a message directly to the queue, and a single Lambda function polls and processes it.

In both cases, the caller gets an immediate "accepted" response and the actual update happens after, decoupled from the request.

Run the integration tests with:

```bash
cd userprofile
source venv/bin/activate
export USERS_STACK_NAME=<users-stack-name>
export USERPROFILE_STACK_NAME=ws-serverless-patterns-userprofile
pytest tests/integration -v
```

## Deploying a Module

Each module folder is a self-contained AWS SAM project:

```bash
cd <module-folder>
sam build
sam deploy --guided --stack-name <stack-name>
```

`module3` (`orders`) and `module4` (`userprofile`) require the `UserPool` parameter, taken from the Users stack's `UserPool` output.

## Cleanup

Each stack can be deleted independently with the AWS SAM CLI, in reverse order of dependency (delete `orders` and `userprofile` before `users`):

```bash
sam delete --stack-name <stack-name>
```

## Security

AWS credentials and access keys are not stored in this repository. Deployment uses the AWS CLI's configured credentials (`aws configure`) on the local machine.

## Result

The project demonstrates the core serverless architecture patterns end to end: a synchronous CRUD API with idempotent writes (Orders), and an asynchronous, event-driven API using both EventBridge and SQS (User Profile), all authenticated against a shared Cognito user pool and observable through structured logs, custom metrics, and X-Ray tracing.
