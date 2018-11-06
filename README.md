# Lambda training app

Sample project just for the purpose of learning AWS Lambdas and DynamoDB

## Creating New Project

### Requirements
* Linux/Mac
* node.js 8.10.*

### Install Dependencies
in project folder run:


To initialize new javascript project (leave all the options to defaults)
```
npm init
```

Install Serverless framework for developing basic Lambdas
```
npm install serverless -g
npm install serverless --save-dev
npm install serverless-offline --save-dev
```

DynamoDB support
```
npm install serverless-dynamodb-local --save-dev 
```

AWS SDK for node.js
```
npm install aws-sdk
```

## Create New Serverless Project
```
serverless create --template aws-nodejs
```
update generated ``serverless.yml`` config file 
and add follwing lines at the end of the file to support:

* offline serverless
* local DynamoDB

```
plugins:
  - serverless-dynamodb-local
  - serverless-offline
```
Install new DynamoDB through serverless:

```
serverless dynamodb install
```

## Run Serverless offline
```
./node_modules/serverless/bin/serverless offline
```
output :
```
Serverless: Starting Offline: dev/us-east-1.

Serverless: Routes for hello:
Serverless: (none)

Serverless: Offline listening on http://localhost:3000
```
Access ``http://localhost:3000`` from your browser.You should see:
```
{
  "statusCode": 404,
  "error": "Serverless-offline: route not found.",
  "currentRoute": "get - /",
  "existingRoutes": []
}
```

## Configure DynamoDB
Update generated serverless.yml with the following changes:


Update ``provider`` section by adding :
```
environment: #define env variable that can be referenced in the source code and configuration files
    TABLE_NAME: ${self:service}-${self:provider.stage}-ExampleTable
  iamRoleStatements: #define basic access rules for our dynamo DB
    - Effect: Allow
      Action:
        - dynamodb:GetItem
        - dynamodb:PutItem
      Resource: arn:aws:dynamodb:*:*:* #allow the operations listed above on all tables
```

Add ``custom`` section to define our offline DynamoDB :
```
custom:
  dynamodb:
      start:
        port: 8000
        inMemory: true
        migrate: true # create tables on start
```

Add new resource :
```
resources:
  Resources:
    ExampleDynamoDB: #just a resource name
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.TABLE_NAME}
        AttributeDefinitions:
          - AttributeName: uuid
            AttributeType: S # (S/N/B(binary))
        KeySchema:
          - AttributeName: uuid
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 5
          WriteCapacityUnits: 5
```

Try running the serverless project with DynamoDB:
```
serverless offline start
```
output(starting DynamoDB):
```
Dynamodb Local Started, Visit: http://localhost:8000/shell
Serverless: DynamoDB - created table aws-nodejs-training-dev-ExampleTable
Serverless: Starting Offline: dev/us-east-2.
.....
.....
```
## Connecting to DynamoDB
Replace code of ``handler.js`` with this example Lambda Handler:
```
'use strict';

var AWS = require('aws-sdk');

module.exports.testdb = async (event, context) => {

    // use local DynamoDB for offline mode
    let dbConfiguration = process.env.IS_OFFLINE ? {region: 'localhost', endpoint: 'http://localhost:8000'} : {};
    let dynamo = new AWS.DynamoDB(dbConfiguration);

    //check DB access
    return new Promise((resolve,reject) => {
        dynamo.listTables({}, function (err, data) {
            if (err) {
                reject({
                    statusCode: 500,
                    body: JSON.stringify({
                        message: 'Unable to connect to DynamoDB',
                        error: err
                    })
                });
            } else {
                resolve({
                    statusCode: 200,
                    body: JSON.stringify({
                        message: 'DynamoDB Connected',
                        connection: dynamo
                    }),
                });
            }
        });
    });
};
```

Add newly created handler to serverless.yml
```
functions:
  example:
    handler: handler.testdb
    events:
      - http:
          path: testdb
          method: get
          cors: true
          private: false
```

## Deployment

1. install AWS CLI - https://docs.aws.amazon.com/cli/latest/userguide/installing.html
2. for Linux/Unix systems set secret access env variable ``export AWS_SECRET_ACCESS_KEY=<your secret>``
3. for Linux/Unix systems set access env variable ``export AWS_ACCESS_KEY_ID=<your key>``

To deploy run :
```
serverless deploy
```

## Result
See folder ``result`` for the final result.

## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License
[MIT](https://choosealicense.com/licenses/mit/)
