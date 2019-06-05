# Introduction
AWS CloudFormation allows us to manage and define AWS stacks through code.

In this tutorial, you will learn how to create a CloudFormation template from scratch.

# Creating the bucket that will contain our function code
We need to create a deployment bucket that will be storing our functions code
`aws s3 mb s3://my-dummy-cloud-formation (optional --region=<region i.e. us-east-1>)`

**Important:** The S3 bucket must be in the same region that the Stack will be deployed to.

# Preparing the workspace
- Create a directory for your project
- Install [CloudFormation](https://marketplace.visualstudio.com/items?itemName=aws-scripting-guy.cform) plugin for VSCode
- Install AWS Cli
- Setup the AWS Cli with your credentials

# Populating CF template
1. In order to get started, we will need to create a new file in the root of the workspace we just created above
2. Update file type to json (ctrl + k + m) -> YAML
3. Type `start` in the new file and press enter
You will see that now we have an empty template
4. Save the file as `cf-template.yml`

## Provisioning Lambda Function
There are a few resources that we will need in order to provision our Lambda function, below are described.

### Creating function code
A Lambda function is usually defined together with an IAM role for execution and a trigger.

Let's create a new directory for the functions: `src/functions` and a new file within that directory named `HelloWorld.js`

we will paste this very simple handler inside:
```js
'use strict';

exports.handler = function(event, context, callback) {
    callback(null, "Hello World!!!!")
}
```

Now that we have a function created, we will move to the `cf-template.yml` file in order to add the function to our resources

### Creating Lambda IAM role
First of all, we will define an **IAM role** for our Lambda function, this role is usually called `Lambda Execution Role` because it determines what actions the function is able to perform. For example, being able to read from S3, or writing to a DynamoDB table. By default, we will usually add the permissions for writing logs to CloudWatch as a good practice.

```yml
...
Resources:
  lambdaExecutionRole:
        Type: AWS::IAM::Role
        Properties:
            AssumeRolePolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Effect: "Allow"
                      Principal:
                          Service:
                              - lambda.amazonaws.com
                      Action:
                          - sts:AssumeRole
            Path: "/"
            Policies:
                - PolicyName: "booster-function-logs"
                  PolicyDocument:
                      Version: "2012-10-17"
                      Statement:
                          - Effect: "Allow"
                            Action:
                                - "Logs:*"
                            Resource: "arn:aws:logs:*:*:*"
```

### Creating Log Group for our Lambda function
In order to gather the logs for a function, we will create a `LogGroup`

```yml
...
Resources:
...
  helloWorldLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: "/aws/lambda/booster-cf-helloWorld"
```

### Creating Lambda function
Then, we will add the function, which depends on the role above created in order to being able to log into CloudWatch and a log group able to gather all the logs related to this function

```yml
...
Resources:
...
    helloWorldFunction:
        Type: AWS::Lambda::Function
        Properties:
            Handler: src/functions/HelloWorld.handler
            Role:
                Fn::GetAtt: lambdaExecutionRole.Arn
            Runtime: "nodejs8.10"
            MemorySize: 128
            Timeout: 5
            Environment:
                Variables:
                    "test": "something dummy"
            Description: "Another Lambda function"
        DependsOn:
            - helloWorldLogGroup
            - lambdaExecutionRole
...
```

### Creating Lambda - Api Gateway Permission
We want this function to be able to be triggered by an HTTP Request. In order to do so, we need an Api Gateway able to hit our Lambda Function. As a result, we need to let the Api Gateway access our function by creating a Lambda Permission.

```yml
...
Resources:
...
  HelloWorldApiPermission:
        Type: AWS::Lambda::Permission
        Properties:
            Action: "lambda:invokeFunction"
            FunctionName:
                Fn::GetAtt: helloWorldFunction.Arn
            Principal: "apigateway.amazonaws.com"
            SourceArn:
                Fn::Join:
                    - ""
                    - - "arn:aws:execute-api:"
                      - Ref: "AWS::Region"
                      - ":"
                      - Ref: "AWS::AccountId"
                      - ":"
                      - Ref: RestApiGateway
                      - "/*"
```

## Creating Lambda trigger resources
Now that we have the function and role defined, we should also define a trigger for the function, for example, an HTTP request through API Gateway

### Api Gateway RestApi
A RestApi is the first thing to be created and will be in charge of containing all the necessary API Gateway resources to expose our Lambda function

```yml
...
Resources:
...
  RestApiGateway:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: "booster-cf-api"
            Description: "Booster CF Api Gateway"
```

### Api Gateway Resource
The resource defines the path for our endpoint

```yml
...
Resources:
...
  HelloWorldResource:
        Type: AWS::ApiGateway::Resource
        Properties:
            RestApiId:
                Ref: RestApiGateway
            ParentId:
                Fn::GetAtt: RestApiGateway.RootResourceId
            PathPart: "hello"
```

### Api Gateway Method
For the function we have created, we will only need a GET method. This method will not perform any Auth.

```yml
...
Resources:
...
  HelloWorldGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            RestApiId:
                Ref: RestApiGateway
            ResourceId:
                Ref: HelloWorldResource
            HttpMethod: GET
            AuthorizationType: NONE
            Integration:
                IntegrationHttpMethod: POST
                Type: AWS_PROXY
                Uri:
                    Fn::Sub:
                        - "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${lambdaArn}/invocations"
                        - lambdaArn:
                              Fn::GetAtt: helloWorldFunction.Arn
```

### Creating a stage for our Api Gateway
The `Api Gateway Deployment` resource will allow us to reuse the same template for different stages, for example, `dev, st, uat, prod` by simply changing the StageName and StackName

```yml
...
Resources:
...
  ApiGatewayDeployment:
        Type: AWS::ApiGateway::Deployment
        DependsOn:
            - HelloWorldGetMethod
        Properties:
            RestApiId:
                Ref: RestApiGateway
            StageName:
                Ref: StageParameter
```

## Generating outputs
We might want to know the ARN of our Lambda functions or the Api Gateway endpoint. `Outputs` can be defined as follow:

```yml
...
Description: ...
...
Resources: ...
...
Outputs:
  apiGatewayInvokeURL:
    Value:
      Fn::Sub: https://${RestApiGateway}.execute-api.${AWS::Region}.amazonaws.com/dev
  lambdaArn:
    Value:
      Fn::GetAtt: helloWorldFunction.Arn
...
```

# Packaging code and generating deployment template
When packaging through `aws cloudformation package...` The code will be uploaded to S3 and a new template file containing the function Code S3 bucket and S3 object key will be returned

`aws cloudformation package --template cf-template.yml --s3-bucket=my-dummy-cloud-formation --output-template-file final-cf-template.yml`

# Deploying stack
When deploying a stack, we need to specify the location of our CloudFormation template file, the stack name, and optionally but recommended tags that will be inherited by the resources created to help managing resources.

`aws cloudformation deploy --template-file final-cf-template.yml.yml --stack-name booster-cf-functions --capabilities CAPABILITY_IAM --tags project=booster contact=tai@theagilemonkeys.com stage=dev`

# Updating function code
If you want to update the code of your functions, you would need to `package` the CloudFormation template first and then perform a new deployment.
