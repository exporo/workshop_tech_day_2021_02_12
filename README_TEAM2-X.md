# Get Lost In Translation with AWS EventBridge
## Team 2-X - [Account: 366497248108]
### Requirements

[Node.js](https://nodejs.org/) v12+.
[Serverless](https://www.npmjs.com/package/serverless)

```sh
$ npm install -g serverless
```

### Create boilerplate

```sh
$ serverless create --template aws-nodejs-typescript
$ npm install
```
### Install Dependencies
[AWS SDK](https://www.npmjs.com/package/aws-sdk)

```sh
$ npm install aws-sdk --save
```

### Update - Serverless.ts

```typescript
import type { AWS } from '@serverless/typescript';

import { hello } from './src/functions';

const EVENT_SOURCE = 'account2';
const REGION = 'us-east-1';
const TEAM_NAME = {{}};//alphanumeric, dash
const serverlessConfiguration: AWS = {
  service: TEAM_NAME,
  frameworkVersion: '2',
  custom: {
    webpack: {
      webpackConfig: './webpack.config.js',
      includeModules: true
    }
  },
  plugins: ['serverless-webpack'],
  provider: {
    name: 'aws',
    region: REGION,
    profile: 'exporo-tech-workshop-2',
    runtime: 'nodejs12.x',
    apiGateway: {
      minimumCompressionSize: 1024,
      shouldStartNameWithService: true,
    },
    environment: {
      AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1',
      EVENT_SOURCE,
    },
    lambdaHashingVersion: '20201221',
  },
  functions: { hello }
}

module.exports = serverlessConfiguration;

```

### Update - src/functions/hello/schema.ts
```typescript
export default {
  type: "object",
  properties: {
    message: { type: 'string' },
    target: { type: 'string' },
  },
  required: ['message', 'target']
} as const;
```

### Create - src/libs/eventBridge.ts
```typescript
import { EventBridge } from 'aws-sdk';
import { FromSchema } from 'json-schema-to-ts';
import schema from '../functions/hello/schema';

const eventBridge = new EventBridge();

export enum detailTypes {
  UPDATED_MESSAGE = 'UPDATED_MESSAGE',
}

export const putEventUpdatedMessage = (message: FromSchema<typeof schema>) => {
  return eventBridge.putEvents({
    Entries: [
      {
        Source: process.env.EVENT_SOURCE,
        DetailType: detailTypes.UPDATED_MESSAGE,
        Detail: JSON.stringify(message),
      }
    ]
  }).promise();
}
```

### Update - src/functions/hello/handler.ts
```typescript
import 'source-map-support/register';

import type { ValidatedEventAPIGatewayProxyEvent } from '@libs/apiGateway';
import { formatJSONResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';

import schema from './schema';
import { putEventUpdatedMessage } from '@libs/eventBridge';

const hello: ValidatedEventAPIGatewayProxyEvent<typeof schema> = async (event) => {
  const response = await putEventUpdatedMessage(event.body);
  return formatJSONResponse({
    message: `Hello ${event.body.message}, welcome to the exciting Serverless world!`,
    response,
  });
}

export const main = middyfy(hello);
```

### Create Rule to filter and log events
- Go to EventBridge page
- Go to Events - Rules - Create Rule
- Input rule name, e.g `log-events-team-2`
- On `Define Patterns` select `Event pattern`
- Select `Custom pattern`
- In the `Event pattern` textarea add:
  ```json
  {
      "detail-type": ["TRANSFORMED_MESSAGE"]
      "detail": {
          "target": ["YOUR_TEAM_NAME_HERE"]
      }
  }
  ```
 - Click on `Save`
 - On `Select targets` - `Target` dropdown list select `CloudWatch log group`
 - On `/aws/events/` input e.g `log-events-team-2`
 - Click on `Create`

### Deploy & Pray
```sh
$ sls deploy
```

### Receive event log
  - Go to cloudwatch
  - Find log-events-team-x

> Message will arrive with asterisks, e.g `once **** a **** when pigs drank ****`, fill the asterisks with your understanding and brings new words to the message, then use the new message to request and target the next team.

### Request
 - Copy endpoints: POST URL from deploy logs
 - Post request with
 ```json
   {
     "message": "NEW_MESSAGE",
     "target": "TARGET_TEAM_NAME"
   }
```