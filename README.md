# Get Lost In Translation with AWS EventBridge
## Team 1 - [Account: 136295262843
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
[serverless-events-cross-organization](https://github.com/c4e/serverless-events-cross-organization)
[AWS SDK](https://www.npmjs.com/package/aws-sdk)
[lodash](https://lodash.com/docs/4.17.15)

```sh
$ npm install serverless-events-cross-organization --save
$ npm install aws-sdk --save
$ npm install lodash --save
```

### Update - Serverless.ts

```typescript
import type { AWS } from '@serverless/typescript';

import { hello } from './src/functions';

const STRATEGIES_DEV_ACCOUNT_ID = '366497248108';
const ORGANIZATION_ID = 'o-z79ijhymgf';
const EVENT_SOURCE = 'exporo-team1';
const REGION = 'us-east-1';
const serverlessConfiguration: AWS = {
  service: EVENT_SOURCE,
  frameworkVersion: '2',
  custom: {
    webpack: {
      webpackConfig: './webpack.config.js',
      includeModules: true
    },
    eventBridgeCrossOrganization: {
      sendEvents: [
        {
          targetAccountId: STRATEGIES_DEV_ACCOUNT_ID,
          region: REGION,
          pattern: {
            "source": [EVENT_SOURCE],
            "detail-type": ['TRANSFORMED_MESSAGE'],
          }
        }
      ],
      receiveEvents: {
        statementId: `${EVENT_SOURCE}_statementID`,
        organizationId: ORGANIZATION_ID,
      }
    }
  },
  plugins: ['serverless-webpack', 'serverless-events-cross-organization'],
  provider: {
    name: 'aws',
    runtime: 'nodejs12.x',
    region: REGION,
    profile: 'exporo-tech-workshop-1',
    apiGateway: {
      minimumCompressionSize: 1024,
      shouldStartNameWithService: true,
    },
    environment: {
      AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1',
      EVENT_SOURCE,
      REGION
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
### Create EventBridge Schema
- Go to EventBridge page
- Go to Schema registry - Schemas section
- Create registry `tech-workshop`
- Create schema `exporo@glit-message`
  - Discovery from JSON
    ```json
    {message: "message", "target": "target"}
    ```

Use Visual Studio Code `AWS Toolkit` plugin to download Schema to the src project or download code binding from aws console and paste the folder `schema` into `src/` folder

### Create - src/libs/eventBridge.ts
```typescript
import { EventBridge } from 'aws-sdk';
import { FromSchema } from 'json-schema-to-ts';
import schema from '../functions/hello/schema';
import { Event as MessageEvent } from '../schema/exporo/glit_message/Event';

const eventBridge = new EventBridge();

export enum detailTypes {
  CREATED_MESSAGE = 'CREATED_MESSAGE',
  UPDATED_MESSAGE = 'UPDATED_MESSAGE',
  TRANSFORMED_MESSAGE = 'TRANSFORMED_MESSAGE',
}

export const createMessage = (message: FromSchema<typeof schema>) => {
  return eventBridge.putEvents({
    Entries: [
      {
        Source: process.env.EVENT_SOURCE,
        DetailType: detailTypes.CREATED_MESSAGE,
        Detail: JSON.stringify(message),
      }
    ]
  }).promise();
}

export const putEventTransformedMessage = (message: MessageEvent) => {
  return eventBridge.putEvents({
    Entries: [
      {
        Source: process.env.EVENT_SOURCE,
        DetailType: detailTypes.TRANSFORMED_MESSAGE,
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
import { createMessage } from '@libs/eventBridge';

const hello: ValidatedEventAPIGatewayProxyEvent<typeof schema> = async (event) => {
  const response = await createMessage(event.body);
  return formatJSONResponse({
    message: `Hello ${event.body.message}, welcome to the exciting Serverless world!`,
    response,
  });
}

export const main = middyfy(hello);
```

### Create - src/functions/transformMessage/index.ts
```typescript
import { detailTypes } from "../../libs/eventBridge";

export default {
  handler: `${__dirname.split(process.cwd())[1].substring(1)}/handler.main`,
  events: [
    {
      eventBridge: {
        pattern: {
          "detail-type": [detailTypes.CREATED_MESSAGE, detailTypes.UPDATED_MESSAGE]
        }
      }
    }
  ]
}
```
## Update - src/functions/index.ts
```typescript
export { default as hello } from './hello';
export { default as transformMessage } from './transformMessage';
```
## Update - serverless.ts
```typescript
import { hello, transformMessage } from './src/functions';
...
functions: { hello, transformMessage }
}

module.exports = serverlessConfiguration
```
### Create - src/functions/transformMessage/handler.ts
```typescript
import 'source-map-support/register';

import { middyfy } from '@libs/lambda';
import { EventBridgeHandler } from 'aws-lambda';
import { Event as MessageEvent } from '../../schema/exporo/glit_message/Event';
import { detailTypes, putEventTransformedMessage } from '@libs/eventBridge';
import * as _ from 'lodash';

const transformMessage: EventBridgeHandler<detailTypes, MessageEvent, void> = async (event) => {
  const percetWordsToRemove = 30;
  const words = event.detail.message.split(' ');
  const wordsToRemove = Math.round(percetWordsToRemove * words.length / 100);
  const indexWordsToRemove = _.sampleSize(_.range(0, words.length-1), wordsToRemove);

  const lostInTranslactionMessage = words.map((word, index) => {
    if(indexWordsToRemove.includes(index)) word = _.map(word, () => '*').join('');

    return word
  }).join(' ');

  await putEventTransformedMessage({ ...event.detail, message: lostInTranslactionMessage});
}

export const main = middyfy(transformMessage);
```

### Deploy & Pray
```sh
$ sls deploy
```
