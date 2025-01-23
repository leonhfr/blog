---
title: Dynamodb Update Expressions
description: Description
date: 2021-10-11
author: leon h
draft: true
favorite: false
tags:
  - aws
  - dynamodb
---

Using update expressions to update maps in DynamoDB

<!--more-->

```typescript
// schema.ts
import {
  attribute,
  hashKey,
  rangeKey,
  table,
} from '@aws/dynamodb-data-mapper-annotations';

@table('example-table')
export class ExampleSchema {
  @hashKey({ type: 'String' })
  pk: string;
  @rangeKey({ type: 'Number' })
  sk: number;
  @attribute({ memberType: { type: 'Number' } })
  exampleMap: Map<string, number>;
}
```

```typescript
// mapperFormatter.ts
import {
  AttributePath,
  FunctionExpression,
  UpdateExpression,
} from '@aws/dynamodb-expressions';

export function formatUpsertSetup(): UpdateExpression {
  const attributePath = new AttributePath('exampleMap');
  const expression = new UpdateExpression();
  expression.set(
    attributePath,
    new FunctionExpression('if_not_exists', attributePath, new Map())
  );
  return expression;
}

export function formatUpsert(input: {
  [key: string]: number;
}): UpdateExpression {
  const expression = new UpdateExpression();
  Object.entries(input).forEach(([key, value]) =>
    expression.add(new AttributePath(`exampleMap.${key}`), value)
  );
  return expression;
}
```

```typescript
// upsert.ts
import { DataMapper } from '@aws/dynamodb-data-mapper';
import DynamoDB = require('aws-sdk/clients/dynamodb');

const mapper = new DataMapper({
  client: new DynamoDB({ region: 'us-east-1' }),
});

async function upsert(input: ExampleSchema): Promise<ExampleSchema> {
  const primaryKey = { pk: input.pk, sk: input.sk };
  const setupExpression = formatUpsertSetup();
  const upsertExpression = formatUpsert(input.exampleMap);

  await mapper.executeUpdateExpression(
    setupExpression,
    primaryKey,
    ExampleSchema
  );
  return await mapper.executeUpdateExpression(
    upsertExpression,
    primaryKey,
    ExampleSchema
  );
}
```

<!--more-->

https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.ConditionalUpdate

https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.UpdateExpressions.html

https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.OperatorsAndFunctions.html

https://www.alexdebrie.com/posts/dynamodb-condition-expressions/

https://aws.amazon.com/dynamodb/pricing/

https://aws.amazon.com/dynamodb/pricing/on-demand/

https://www.reddit.com/r/aws/comments/a8xda2/dynamodb_conditional_put_cost/
