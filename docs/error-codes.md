---
title: "Error Codes"
sidebar_position: 3
---

### API Gateway does not supported computed property names

__Error Code__: Functionless(10010)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Computed Property Names are not supported in API Gateway.

For example:
```ts
new AwsMethod(
  ($input) => $AWS.DynamoDB.GetItem({
    Table: table,
    // invalid, all property names must be literals
    [computedProperty]: prop
  })
);
```

To workaround, be sure to only use literal property names.

___

### API Gateway does not support spread assignment expressions

__Error Code__: Functionless(10011)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Due to limitations in API Gateway's VTL engine (no $util.toJson, for example)
it is not possible to fully support spread expressions.

For example:
```ts
new AwsMethod(
  () => {},
  ($input) => ({
    hello: "world",
    ...$input.data
  })
);
```

To workaround the limitation, explicitly specify each property.

```ts
new AwsMethod(
  () => {},
  ($input) => ({
    hello: "world",
    propA: $input.data.propA,
    propB: $input.data.propB,
  })
);
```

___

### API gateway response mapping template cannot call integration

__Error Code__: Functionless(10013)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Code running within an API Gateway's response mapping template must not attempt
to call any integration. It can only perform data transformation.

```ts
new AwsMethod(
  ...,
  () => {
    // INVALID!     return $AWS.DynamoDB.GetItem({
      Table: table,
      ...
    });
  }
)
```

To workaround, make sure to only call an integration from within the `request` mapping function.

___

### ApiGateway Unsupported Reference

__Error Code__: Functionless(10029)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

ApiGateway supports the use of many integrations like Function and StepFunction.

Other references to data outside of of the AwsMethod are not supported.

```ts
const myObject = new Object();
const func = new Function(this, 'func', async () => { ... });
new AwsMethod(..., async () => {
   // invalid
   myObject;
   // invalid
   $util.autoUlid();
   // valid
   return func();
}, ...);
```

___

### AppSync Unsupported Reference

__Error Code__: Functionless(10028)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

AppSync supports the use of many integrations like Function and StepFunction.
It also supports special utility functions through $util.

Other references to data outside of of the AppsyncResolver or AppsyncField are not supported.

```ts
const myObject = new Object();
const func = new Function(this, 'func', async () => { ... });
new AppsyncResolver(..., async () => {
   // invalid
   myObject;
   // valid
   $util.autoUlid();
   // valid
   return func();
});
```

___

### Appsync Integration invocations must be unidirectional and defined statically

__Error Code__: Functionless(10020)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Appsync Integration invocations must be unidirectional and defined statically.

As stated in the [AppSync Pipeline Resolvers Documents](https://docs.aws.amazon.com/appsync/latest/devguide/pipeline-resolvers.html):

> Pipeline resolver execution flow is unidirectional and defined statically on the resolver.

```ts
const func = new Function(stack, 'id', async () => {});

new AppsyncResolver(..., async ($context) => {
   // valid
   const f = await func();
   // valid
   await func();
   if($context.arguments.value) {
      // invalid       await func();
   }
   while($context.arguments.value) {
      // invalid
      await func();
   }
   // valid
   return func();
});
```

Workaround:

One workaround would be to invoke a `Function` (or ExpressStepFunction) which handles the conditional parts of the workflow.

The result of this example would be to call the `conditionalFunc` statically and call `func` conditionally.

```ts
const func = new Function(stack, 'id', async () => {});

const conditionalFunc = new Function(stack, 'id', async (value) => {
   if(value) {
      return func();
   }
   return null;
})

new AppsyncResolver(..., async ($context) => {
   const conditionalResult = await conditionalFunc();
   return conditionalResult;
});
```

___

### Argument must be an inline Function

__Error Code__: Functionless(10002)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

The argument must be an inline Function.

```ts
const func = () => {
  // ..
}
// invalid new Function(this, id, func);
```

To fix, inline the `func` implementation.

```ts
// option 1 new Function(this, id, async () => { .. });

// option 2 new Function(this, id, async function () { .. });
```

___

### Arrays of Integration must be immediately wrapped in Promise.all

__Error Code__: Functionless(10017)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Arrays of integrations within StepFunction, ExpressStepFunction must be immediately wrapped in Promise.all.

These services support concurrent invocation, but do not support manipulation of the references.

```ts
const func = new Function<string, string>(stack, 'func', async (input) => { return "hi" });

new StepFunction(stack, 'sfn, async () => {
   // invalid
   const f = [1,2].map(await () => func());
   // valid
   const ff = await Promise.all([1,2].map(await () => func()));
   // valid
   return Promise.all([1,2].map(await () => func()));
});
```

Some Resources like Function support synchronous or asynchronous handing of Integrations.

```ts
new Function(stack, 'func2', async () => {
   // valid
   const f = [1,2].map(await () => func())
   const f2 = [1,2].map(await () => func());;
   return Promise.all([...f, ...f2]);
});
```

> In the case of Function, any valid NodeJS `Promise` feature is supported.

___

### AwsMethod request must have exactly one integration call

__Error Code__: Functionless(10003)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

When using the AwsMethod, the `request` argument must be a function
with exactly one integration call.

```ts
new AwsMethod(
  {
    httpMethod: "GET",
    resource: api.root
  },
  ($input) => {
    return $AWS.DynamoDB.GetItem({ .. });
  },
  // etc.
)
```

___

### Cannot perform all arithmetic on variables in Step Function

__Error Code__: Functionless(10000)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Step Functions can only preform limited arithmetic operations.

Add (`+`), Subtract (`-`) and their duratives (`+=`, `++`) are supported, but operations like multiply (`*`) and mod (`%`) are not.

```ts
// ok
new StepFunction(scope, id, () => 1 + 2);

// ok
new StepFunction(scope, id, (input: { num: number }) => input.number + 1);

// illegal!
new StepFunction(scope, id, (input: { num: number }) => input.number * 2);
```

To workaround, use a `Function` to implement the arithmetic expression. Be aware that this comes with added cost and operational risk.

```ts
const mult = new Function(scope, "add", (input: { a: number, b: number }) => input.a * input.b);

new StepFunction(scope, id, async (input: { num: number }) => {
  await mult({a: input.number, b: 1});
});
```

___

### Cannot use infrastructure Resource in Function closure

__Error Code__: Functionless(10009)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Cannot use Infrastructure resource in Function closure.

The `.resource` property of `Function`, `StepFunction`, `ExpressStepFunction`, `EventBus`, and `Table` are not available
in Native Function Closures.

```ts
const table = new Table(this, 'table', { ... });
new Function(this, 'func', async () => {
   // valid use of a Table
   const $AWS.DynamoDB.GetItem({
       Table: table,
       ...
   })
   // invalid    const index = table.resource.tableStreamArn;
});
```

Workaround 1 
In many cases, common properties are available on the Functionless Resource, for example:

```ts
const table = new Table(this, 'table', { ... });
new Function(this, 'func', async () => {
   const tableArn = table.tableArn;
});
```

Workaround 2 
For some properties, referencing to a variable outside of the closure will work.

```ts
const table = new Table(this, 'table', { ... });
const tableStreamArn = table.resource.tableStreamArn;
new Function(this, 'func', async () => {
   const tableStreamArn = tableStreamArn;
});
```

Workaround 3 
Finally, if none of the above work, the lambda environment variables can be used.

```ts
const table = new Table(this, 'table', { ... });
new Function(this, 'func', {
   environment: {
      STREAM_ARN: table.resource.tableStreamArn
   }
}, async () => {
   const tableStreamArn = process.env.STREAM_ARN;
});
```

___

### Classes are not yet supported by Functionless

__Error Code__: Functionless(10031)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Classes, methods and private identifiers are not yet supported by Functionless.

To workaround, use Functions.

```ts
function foo () { .. }
const foo = () => { .. }
```

**`See`**

https://github.com/functionless/functionless/issues/362

___

### Event Bridge does not support a truthy comparison

__Error Code__: Functionless(10032)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

___

### EventBus Input Transformers do not support integrations

__Error Code__: Functionless(10014)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

EventBus Input Transformers do not support Integrations.

```ts
const func = new Function<string, string>(stack, 'func', async (input) => {
   return axios.get(input);
})

// invalid
new EventBus(stack, 'bus').all().map(async event => ({ html: await func(event.detail.url) }));
```

Workaround 
```ts
const func = new Function<string, string>(stack, 'func', async (input) => {
   return `transform${input}`;
})

// valid
new EventBus(stack, 'bus').all().pipe(new Function(stack, 'webpuller', async (event) => ({
   html: await func(event.detail.url)
})));
```

___

### EventBus Rules do not support integrations

__Error Code__: Functionless(10015)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

EventBus Rules do not support Integrations.

```ts
const validate = new Function<string, string>(stack, 'validate', async (input) => {
   return axios.get(`https://mydomain/validate/${input}`);
})

// invalid
new EventBus(stack, 'bus').when(async (event) => await validate(input.detail.value)).pipe(...);
```

Workaround 
```ts
// valid
new EventBus(stack, 'bus').all().pipe(new Function(stack, 'webpuller', async (event) => {
   if(await validate(event.source)) {
      ...
   }
});
```

___

### Expected an object literal

__Error Code__: Functionless(10012)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Due to limitations in respective Functionless interpreters, it is often a
requirement to specify an object literal instead of a variable reference.

```ts
const input = {
  Table: table,
  Key: {
    // etc.
  }
};
// invalid $AWS.DynamoDB.GetItem(input)
```

To work around, ensure that you specify an object literal.

```ts
$AWS.DynamoDB.GetItem({
  Table: table,
  Key: {
    // etc.
  }
})
```

___

### Function not compiled by Functionless plugin

__Error Code__: Functionless(10001)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

During CDK synth a function was encountered which was not compiled by the Functionless compiler plugin.
This suggests that the plugin was not correctly configured for this project.

Ensure you follow the instructions at https://functionless.org/docs/getting-started.

___

### Function closure serialization was not allowed to complete

__Error Code__: Functionless(10004)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Lambda Function closure synthesis runs async, but CDK does not normally support async.

In order for the synthesis to complete successfully
1. Use autoSynth `new App({ authSynth: true })` or `new App()` with the CDK Cli (`cdk synth`)
2. Use `await asyncSynth(app)` exported from Functionless in place of `app.synth()`
3. Manually await on the closure serializer promises `await Promise.all(Function.promises)`

https://github.com/functionless/functionless/issues/128

___

### Incorrect state machine type imported

__Error Code__: Functionless(10006)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Incorrect State Machine Type Imported

Functionless StepFunctions are separated into ExpressStepFunction and StepFunction
based on being aws_stepfunctions.StateMachineType.EXPRESS or aws_stepfunctions.StateMachineType.STANDARD
respectively.

In order to ensure correct function of Functionless integrations, the correct import statement must be used.

```ts
const sfn = new aws_stepfunctions.StateMachine(scope, 'standardMachine', {...});
// valid
StateMachine.fromStepFunction(sfn);
// invalid ExpressStateMachine.fromStepFunction(sfn);

const exprSfn = new aws_stepfunctions.StateMachine(scope, 'standardMachine', {
   stateMachineType: aws_stepfunctions.StateMachineType.EXPRESS,
});
// valid
ExpressStateMachine.fromStepFunction(exprSfn);
// invalid StateMachine.fromStepFunction(exprSfn);
```

___

### Integration must be immediately awaited or returned

__Error Code__: Functionless(10016)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Integrations within StepFunction, ExpressStepFunction, AppsyncResolver, and AwsMethod must be immediately awaited or returned.

These services do not support references to asynchronous operations. In order to model an asynchronous call being handled synchronously, all integrations that return promises must be awaited or returned.

```ts
const func = new Function<string, string>(stack, 'func', async (input) => { return "hi" });

new StepFunction(stack, 'sfn', async () => {
   // invalid
   const f = func();
   // valid
   const ff = await func();
   // valid
   return func();
});

new AppsyncResolver(..., async () => {
   // invalid
   const f = func();
   // valid
   const ff = await func();
   // valid
   return func();
})

new AwsMethod(
   ...,
   async () => {
      // invalid
      const f = func();
      // valid
      const ff = await func();
      // valid
      return func();
   }
);
```

Some Resources like Function support synchronous or asynchronous handing of Integrations.

```ts
new Function(stack, 'func2' , async () => {
   // valid
   const f = func();
   // valid
   const ff = await func();
   const fDone = await f;
   // valid
   return func();
});
```

> In the case of Function, any valid NodeJS `Promise` feature is supported.

___

### Invalid Input

__Error Code__: Functionless(10027)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Invalid Input

Generic error code for when the user provided an unexpected input as documented and reflected in the types.

See the error message for more details.

___

### StepFunction throw must be Error or StepFunctionError class

__Error Code__: Functionless(10030)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Errors in Step Functions can only be thrown in one of two ways:

1. by throwing javascript's `Error` class
```ts
throw Error("message");
throw new Error("message");
```
2. by throwing the `StepFunctionError` class
```ts
throw new StepFunctionError("CustomErrorName", { error: "data" })
```

___

### StepFunction invalid filter syntax

__Error Code__: Functionless(10034)
__Error Type__: <span style={{ "background-color": "grey", "padding": "4px" }}>DEPRECATED</span>

DEPRECATED 
___

### StepFunctions Invalid collection access

__Error Code__: Functionless(10025)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Step Functions 
  1. Arrays can be accessed with numbers
  2. Objects can be accessed with strings
  3. Access Elements must be constant values 
```ts
const func = new Function(this, 'func', async () => { return { index: 0, elm: "a" }; });
new StepFunction(stack, 'sfn', () => {
   const obj = { a: "val" };
   // valid
   obj.a;
   // valid
   obj["a"];
   // invalid
   obj[await func().elm];

   const arr = [1];
   // valid
   arr[0]
   // invalid
   arr[await func().index];
});
```

Workaround 
```ts
const accessor = new Function<{ obj: [key: string]: string, acc: string }, string>(
   stack,
   'func',
   async (input) => {
      return input.obj[input.acc];
   }
);

new StepFunction(stack, 'sfn', () => {
   // valid
   await accessor(obj, a);
});
```

___

### StepFunctions calls to EventBus putEvents must use object literals

__Error Code__: Functionless(10023)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Events passed to an EventBus in a StepFunction must be one or more literal objects and may not use the spread (`...`) syntax or computed properties.

```ts
const sfn = new StepFunction(stack, "sfn", async () => {
  const sourceName = "source";
  const event = { source: "lambda", "detail-type": "type", detail: {} };
  // invalid
  await bus.putEvents(event);
  // invalid
  await bus.putEvents({ ...event });
  // invalid
  await bus.putEvents(...[event]);
  // invalid
  await bus.putEvents({
    [sourceName]: "lambda",
    "detail-type": "type",
    detail: {},
  });
  // valid
  await bus.putEvents({
    source: "lambda",
    "detail-type": "type",
    detail: {},
  });
});
```

Workaround 
```ts
const sender = new Function(stack, "sender", async (event) => {
  const sourceName = "source";
  const event = { source: "lambda", "detail-type": "type", detail: {} };
  await bus.putEvents(event); // valid
  await bus.putEvents({ ...event }); // valid
  await bus.putEvents(...[event]); // valid
  // valid
  await bus.putEvents({
    [sourceName]: "lambda",
    "detail-type": "type",
    detail: {},
  });
  // valid
  await bus.putEvents({
    source: "lambda",
    "detail-type": "type",
    detail: {},
  });
});

const sfn = new StepFunction(stack, "sfn", async () => {
  const event = { source: "lambda", "detail-type": "type", detail: {} };
  await sender(event);
});
```

The limitation is due to Step Function's lack of optional or default value retrieval for fields.
Attempting to access a missing field in ASL leads to en error.
This can be fixed using Choice/Conditions to check for the existence of a single field,
but would take all permutations of all optional fields to support optional field at runtime.
Due to this limitation, we currently compute the transformation at compile time using the fields present on the literal object.
For more details and process see: https://github.com/functionless/functionless/issues/101.

___

### StepFunctions does not support destructuring object with rest

__Error Code__: Functionless(10033)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Step Functions does not natively support dynamic object manipulation.

Due to this limitation, `...rest` is not supported when destructuring objects.

```ts
new StepFunction(..., async () => {
   const value = { a: "a", b: "b", c: "c" };
   // valid
   const { a, b } = value;
   // invalid
   const { a, b, ...rest } = value;
});
```

In the above example, we'd expect `rest` to look like `{ c: "c" }`,
but Step Functions Amazon States Language does not natively support a way
to enumerate the fields of an object or delete fields from an object without first
knowing all field names in an object.

Workaround 
If `...rest` is being used as a convince, a work around is to just be explicit.

```ts
new StepFunction(..., async () => {
   const value = { a: "a", b: "b", c: "c" };
   // valid
   const { a, b } = value;
   // invalid
   const rest = { c: value.c };
});
```

Workaround 
Sometimes `...rest` is used as part of the program logic, for example,
if some fields of an object are known while others are not
or to remove known fields from an object.

```ts
const extractAAndB = new Function<
  { a: string; b: string; [key: string]: string },
  { a: string; b: string; rest: { [key: string]: string } }
>(stack, "func", async ({ a, b, ...rest }) => ({ a, b, rest }));
new StepFunction(..., async () => {
   const value = { a: "a", b: "b", c: "c" };
   // valid
   const { a, b, rest } = await extractAAndB(value);
});
```

___

### StepFunctions error cause must be a constant

__Error Code__: Functionless(10024)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

StepFunctions supports throwing errors with causes, however those errors do not support dynamic values.

https://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-fail-state.html

```ts
new StepFunction<{ value: undefined }, void>(this, 'sfn', async (input) => {
   // invalid    throw new Error(input.value);
});
```

Workaround 
```ts
new StepFunction<{ value: undefined }, { error?: string }>(this, 'sfn', async (input) => {
   // valid
   return {
      error: input.value
   }
});
```

___

### StepFunctions mismatched index type.

__Error Code__: Functionless(10035)
__Error Type__: <span style={{ "background-color": "yellow", "color": "black", "padding": "4px" }}>WARN</span>

StepFunctions could fail at runtime with mismatched index types.

Javascript and Typescript support the normalization object property access
and array access when the index is a number or number as a string.

Unfortunately SFN cannot repeat this normalization in an inexpensive way
(ex: checking all property access for correctness at runtime).

The below is all valid typescript.

```ts
new StepFunction(stack, 'sfn', async () => {
   const obj = {1: "value"};
   const arr = [1];

   // invalid    obj[1];

   // invalid    arr["0"];

   // valid
   obj["1"];
   // valid
   arr[0];
});
```

___

### StepFunction property names must be constant

__Error Code__: Functionless(10026)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

StepFunctions does not support dynamic property names

```ts
new StepFunction(stack, 'sfn', async () => {
   const c = "c";
   const {
      a: "valid",
      ["b"]: "valid",
      [c]: "invalid",
      [someMethod()] :"invalid"
   }
});
```
Workaround 
```ts
const assign = new Function<{
   obj: { [key: string]: string },
   key: string, value: string
}>(stack, 'func', async (input) => {
    return {
       ...input.obj,
       [input.key]: input.value
    }
});
new StepFunction(stack, 'sfn', async () => {
   return await assign({}, someMethod(), "value");
});
```

___

### Step Function Retry Invalid Input.

__Error Code__: Functionless(10037)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Step Function Retry Invalid Input.

`$SFN.Retry` can either retry on all errors or be given a constant,
array of literal objects (Retry) that determine which errors to retry on.
The array must have 1 or more literal Retry objects.

```ts
new StepFunction(stack, "sfn", async ()=> {
   // Valid
   await $SFN.retry(async () => func());
   // Valid
   await $SFN.retry([{ ErrorEquals: ["States.Timeout"] }], async () => func());
   // Valid
   await $SFN.retry([{ ErrorEquals: ["States.Timeout"], MaxAttempts: 5, IntervalSeconds: 2, BackoffRate: 0.5 }], async () => func());
   // Valid
   await $SFN.retry([{ ErrorEquals: ["States.Timeout"] }, { ErrorEquals: ["Lambda.TooManyRequestsException"] }], async () => func());

   // Invalid    await $SFN.retry(myRetryObjects, async () => func());
   // Invalid    await $SFN.retry([retryObject], async () => func());
   // Invalid    await $SFN.retry([{ ErrorEquals: [ myErrorType ] }], async () => func());
   // Invalid    await $SFN.retry([], async () => func());
})
```

The limitation is due to Step Function's lack of support for dynamic paths (json paths) for retry configuration.

https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html#error-handling-retrying-after-an-error

___

### Step Functions Arithmetic Only Supports Integers

__Error Code__: Functionless(10038)
__Error Type__: <span style={{ "background-color": "yellow", "color": "black", "padding": "4px" }}>WARN</span>

Step Functions Arithmetic Only Supports Integer

Step Functions only supports integer addition via the `States.MathAdd`.

```ts
new StepFunction(stack, "sfn", async (input: { a: number }) => {
   return 1.5 + input.a;
});
```

If the above machine is given input: `{ a: 0.5 }`, the result will be `1`.

Effectively resulting in: `1.5 + 0.5 === 1`

Workaround:

Do floating point math within a lambda function.

```ts
const floatingPointAdd = new Function(stack, "fn", async (input: { a: number, b: number }) => {
   return input.a + input.b;
});
new StepFunction(stack, "sfn", async (input: { a: number }) => {
   return floatingPointAdd(1.5, input.a);
});
```

___

### Step Functions does not support undefined

__Error Code__: Functionless(10022)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Step Functions does not support undefined

In Step Functions, a property cannot be undefined when assigned to an object or passed into a state.

For consistency, `undefined` is not allowed anytime a value is returned, for example in a ternary expression (`x ? undefined : value`).
The `undefined` literal is allowed for comparison. `if(x === undefined){}`.

```ts
const func = new Function(stack, 'func', () => { return undefined; })
new StepFunction<{ val: string | undefined }, undefined>(stack, 'sfn', async (input) => {
   const v = {
      // invalid       val: input.val
   }

   // invalid
   v ? undefined : "a"

   // invalid, function outputs undefined.
   const output = await func();

   // invalid    return input.val;
});
```

1. Workaround 
   * ```ts
const func = new Function(stack, 'func', () => { return null; })
new StepFunction<{ val: string | null }, null>(stack, 'sfn', async () => {
   const v = {
      // valid
      val: null
   }

   // valid
   v ? null : "a"

   // valid, function outputs undefined.
   const output = await func();

   // valid
   return null;
});
```

2. Workaround 
   * ```ts
const func = new Function(stack, 'func', () => { return undefined; })
new StepFunction<{ val: string | undefined }, null>(stack, 'sfn', async (undefined) => {
   const v = {
      // valid       val: input.val ?? null
   }

   // valid    const output = (await func()) ?? null;

   // valid    return input.val ?? null;
});
```

3. Workaround 
```ts
const func = new Function(stack, 'func', () => { return { val: undefined }; })
new StepFunction<{ val: string | undefined }, null>(stack, 'sfn', async (undefined) => {
   let v;
   if(input.val) {
       v = {
          val: input.val
       }
   } else {
       v = {}
   }

   // valid    const output = await func();
   const val = output ? output : null;

   // valid    if(input.val) {
      return input.val;
   }
   return null;
});
```

___

### Unable to find reference out of application function

__Error Code__: Functionless(10019)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Unable to find reference out of application function.

Functionless processes your application code to find infrastructure references, wire up permissions, build clients, and more.
All infrastructure must be created outside of your application logic closures.
References to those resources (Integrations) must be traceable back to their location outside of your application functions.

```ts
const func = new Function(...);

new Function(stack, 'id', {
   // valid
   func();
   const f = func();
   // valid
   f();
   const a = { f };
   // valid
   a.f();
   // and more
})
```

Functionless attempts to handle all valid typescript referencing scenarios, but some may be missed.

If this error is thrown and the reference should be valid, please [create an issue](https://github.com/functionless/functionless/issues).

___

### Unexpected Error

__Error Code__: Functionless(10005)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Generic error message to denote errors that should not happen and are not the fault of the Functionless library consumer.

Please [report this issue](https://github.com/functionless/functionless/issues).

___

### Unsafe use of secrets

__Error Code__: Functionless(10007)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Unsafe usage of Secrets.

The use of secrets is unsafe or not supported by Functionless.

**`See`**

https://github.com/functionless/functionless/issues/252 to track supported secret patterns.

___

### Unsupported AWS SDK in Resource.

__Error Code__: Functionless(10036)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

The AWS SDK being used in the current resource is unsupported by that resource.

Workaround:

Most of the ASK APIs will work within

___

### Unsupported feature

__Error Code__: Functionless(10021)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Unsupported Feature

Generic error for unsupported features.

See error message provided for more details.

___

### Unsupported use of Promises

__Error Code__: Functionless(10018)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Unsupported use of Promises

In StepFunction and ExpressStepFunction, `Promise.all` can be used to turn an array of promises into a single promise. However, it must directly wrap a method which returns an array of promises.

```ts
const func = new Function(stack, id, async () => {});

new StepFunction(stack, id, async () => {
   const c = [1,2];
   // invalid
   const p = await Promise.all(c);
   // valid
   const t = await Promise.all([1,2].map(i => func()));
});
```

AppsyncResolver and AppsyncField do not support `Promise.all` or arrays of `Promises`. AWS App Sync does not support concurrent or dynamic invocation of integrations.

```ts
const func = new Function(stack, id, async () => {});

new AppsyncResolver(..., async ($context) => {
    // valid     const t = await func();
    const tt = await func();
    // invalid     const arr = await Promise.all([1,2].map(() => func()));
    // invalid     const arr = await Promise.all($context.arguments.arr.map(() => func()));
});
```

___

### Unsupported initialization of Resources in a runtime closure

__Error Code__: Functionless(10008)
__Error Type__: <span style={{ "background-color": "red", "padding": "4px" }}>ERROR</span>

Unsupported initialization of Resources in a Function closure

1. Valid ```ts
const bus = new EventBus(this, 'bus');
const function = new Function(this, 'func', () => {
   bus.putEvents(...);
});
```

2. Invalid ```ts
const function = new Function(this, 'func', () => {
   new EventBus(this, 'bus').putEvents(...);
});
```

3. Invalid ```ts
function bus() {
   return new EventBus(this, 'bus');
}
const function = new Function(this, 'func', () => {
   bus().putEvents(...);
});
```

4. Valid ```ts
const bus = new EventBus(this, 'bus');
function bus() {
   return bus;
}
const function = new Function(this, 'func', () => {
   bus().putEvents(...);
});
```
