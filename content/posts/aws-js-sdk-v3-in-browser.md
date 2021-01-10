---
title: "Using AWS JavaScript SDK v3 in Browsers"
date: 2021-01-10
draft: false
toc: false
images:
tags:
  - aws
  - javascript
  - technical
  - typescript
  - webpack
---

In this article, I will walkthrough the process of incorporating the "new" AWS JavaScript SDK in a web application, particularly how to use it directly in browsers.

I wrote this because I found relevant documentation a bit lacking. Also, the idea can be applied to using TypeScript in browsers in general.

## Introduction to AWS SDK for JavaScript v3

AWS have recently released [Version 3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/) of the SDK for JavaScript, targeting the following issues of its previous version (V2):

- Modularized packages for individual services (ie. separate packages for S3, EC2, etc., kind of like microservices)
- New middleware stack

## Let's help our tools help us

### TypeScript over JavaScript

The most interesting feature to me, is that this new version is written in [TypeScript](https://www.typescriptlang.org/). We all know that it's easy to write type-unsafe and buggy JavaScript due to its dynamic typing system. One component of our final-year design project is in vanilla JavaScript, and people easily get perplexed when looking at the code because they need a lot of time to figure out what certain variables are. For example, suppose that we have a job scheduler:

```javascript
class Scheduler {
  jobQueue;

  constructor(jobs) {
    jobs.forEach((job) => this.jobQueue.push(job));
  }

  addJob(job) {
    this.jobQueue.push(job);
  }

  runJobUntilNone() {
    while (this.jobQueue.length) {
      let job = this.jobQueue.pop();
      job.run();
    }
  }
}
```

Although the code above is simple, what exactly is a "job"? It's pretty intuitive to deduce that `jobQueue` is an array of jobs, but what is a `job`? Is it a string? Probably not since `string` doesn't have a `run()` function. What type is it?

Apart from loose typing, nobody will stop you if you call `addJob` by passing a number, string, or any other type. What's worse, is that you will only catch the error at run time, which could lead to millions of dollars of loss for your company...

TypeScript allows us to add type annotations and any type violation will be caught at compile time by the TypeScript compiler:

```typescript
class Job {
    // some member variables
    constructor(args...){//...}

    run() {// some logic}
}

class Scheduler {
  jobQueue: Job[];

  constructor(jobs: Job[]) {
    jobs.forEach((job) => this.jobQueue.push(job));
  }

  addJob(job: Job) {
    this.jobQueue.push(job);
  }

  runJobUntilNone() {
    while (this.jobQueue.length) {
      let job = this.jobQueue.pop();
      job.run();
    }
  }
}
```

The main difference is that we have "enforced" the type of the elements in `jobQueue` to be of type `Job`, which seems insignificant. However, if we pass any other type into the constructor or `addJob`, TypeScript won't compile due to type errors. Hence, those errors are caught at compile time which will absolutely reduce the chance of run time errors associated with types.

There are many other benefits of TypeScript, especially when it comes to development tooling. I use VSCode which comes with TypeScript support [out of the box](https://code.visualstudio.com/Docs/languages/typescript). If we use TypeScript properly and provide decent JSDoc on top of our code, the power of VSCode will be unleashed even more, greatly enhancing our productivity.

### A note about TypeScript

Browsers only understand JavaScript and cannot directly run TypeScript, so TypeScript needs to be compiled to JavaScript before using it in browsers.

This can be easily done by the [TypeScript compiler](https://code.visualstudio.com/docs/typescript/typescript-compiling) and many other JavaScript module bundlers, such as [webpack](https://webpack.js.org/).

## Using AWS SDK V3 to fetch from S3

### Install TypeScript

We need to install TypeScript as our project development dependency:

```shell
npm install --save-dev typescript @types/node
```

`@types/node` will provide us with type annotations in VSCode for various libraries.

### Configure TypeScript compiler

First, we need to configure our TypeScript compiler for our project in a file named `tsconfig.js`. More information about this file can be found [here](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html). Mine looks like the following:

```json
{
  "compilerOptions": {
    "target": "ES6",
    "module": "ES6",
    "sourceMap": true,
    "declaration": true,
    "declarationDir": "./dist",
    "moduleResolution": "node",
    "typeRoots": ["node_modules/@types"],
    "lib": ["dom"]
  },
  "exclude": ["node_modules"],
  "include": ["src/s3.ts"]
}
```

As you see, we will put our code in `src/s3.ts`. We also want to generate the associated [declaration file](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html) under the `dist` directory. Since we will use `@aws-sdk` node modules, we should tell the compiler to remember to look into `node_modules` to find them by setting `moduleResolution`. Also, the most important options are probably `target` and `module`: `target` specifies what version of JavaScript TypeScript will compile to, and `module` specifies how we use JavaScript modules in our code. `es6` means that we can use the newer `import` syntax.

### Code

The best way to learn something new is to practice. Let's write a JavaScript library to be used in browsers that fetchs a file to a S3 bucket. I will go with the [AWS Cognito Identity Pool](https://docs.aws.amazon.com/cognito/latest/developerguide/identity-pools.html) which gives users temporary access to your AWS services, which, in our case, are fetching files to a S3 bucket.

First, let's install the modules we need for fetching from S3:

```shell
npm install --save @aws-sdk/client-s3 @aws-sdk/client-cognito-identity @aws-sdk/credential-provider-cognito-identity
```

We then import them in our code:

```typescript
import { S3 } from "@aws-sdk/client-s3";
import { CognitoIdentityClient } from "@aws-sdk/client-cognito-identity";
import { fromCognitoIdentityPool } from "@aws-sdk/credential-provider-cognito-identity";
```

Then, we define a function to fetch a file specified by a "key" from S3. We assume that we know the identity pool ID, the region, and the bucket name. We only require the object key from the user:

```typescript
export function fetchFromBucket(key: string) {
  const region = "ca-central-1";

  // Initialize an S3 service with credentials for our identity pool.
  const s3 = new S3({
    region: region,
    credentials: fromCognitoIdentityPool({
      client: new CognitoIdentityClient({ region: region }),
      identityPoolId: identityPoolId,
    }),
  });

  // Fetch and print out the object size
  s3.getObject(
    {
      Bucket: "my-example-bucket",
      Key: key,
    },
    (err, data) => {
      if (err) {
        console.log(`Error when fetching from bucket: ${err.stack}`);
      } else {
        console.log(`Data fetched from bucket. Size: [${data.ContentLength}]`);
      }
    }
  );
}
```

That's our straightforward logic. Our next step is to make it runnable in a browser.

### Bundle with webpack

I mentioned that TypeScript needs to be compiled to JavaScript to run in browsers. Apart from that, we also need to bundle the AWS modules in use with our library. We can achieve this using [webpack](https://webpack.js.org/), which is a bundler that can bundle any web application asset. The short version of what it does is that it will put all necessary code into one file, which we can use in a browser by including it in an HTML file with a `script` tag.

Webpack doesn't understand TypeScript by default, but it has a rich ecosystem and comes with a TypeScript plugin. We also need to bundle JSON files into our code as the AWS SDK uses them, and there's a JSON plugin for that as well. We need to install them as our development dependencies:

```shell
npm install --save-dev webpack webpack-cli ts-loader json-loader
```

We then define a configuration file for webpack, named `webpack.config.js`:

```javascript
const path = require("path");

module.exports = {
  target: "web",
  entry: {
    s3: "./src/s3.ts",
  },
  mode: "development",
  module: {
    rules: [
      {
        test: /\.ts?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
      {
        test: /\.json$/,
        use: "json-loader",
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: [".ts", ".js"],
  },
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].js",
    library: "[name]",
  },
};
```

The configuration file is easy to understand, and the more important ones are:

- `target` is how our code will be used. We target at the browsers, hence "web" is the value. It's also the default value.
- `entry` specifies the entry point of our library code, which is our code to fetch from S3. Webpack will start with this file and construct a dependency graph. It will get all other modules it uses, and also the modules used by those modules, and so on. It then "bundles" them into a single file that contains all the code we need to fetch from S3.
- `mode` affects the formatting of our bundled file. `development` will keep our code in a format that is easy to develop. In contrast, `production` will minify our code completely by removing all comments, whitespaces, newlines, etc.
- `module` specifies the plugins. We are compiling TypeScript to JavaScript and also including JSON files, hence we have two rules for them respectively.
- `output` apparently specifies the bundled file. Because it's a library, we need to specify the value for `output.library`. `[name]` maps to the keys in `entry`, so our bundled file will be `./dist/s3.js` and the function `fetchFromBucket` is under `s3`. To use it, we simply call `s3.fetchFromBucket()`.

Finally, we can add a simple `build` script in `package.json`:

```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack"
  },
```

To build/bundle our code, simply run

```shell
npm run build
# we can also run "webpack" directly.
```

This generates `./dist/s3.js` which we can directly import into browsers!

```html
<script src="s3.js">
<script src="index.js">
```

and in `index.js`, we can call our function to fetch an object from S3:

```javascript
s3.fetchFromBucket("example_object_key");
```

## One more thing

This seems a rather long process. It takes time initially but the rest of the team will easily benefit from this workflow. Also, I learned about what module bundlers could do and how to create JavaScript libraries for various platforms.

Although I'm not into front-end web development, I've been amazed by its rich tooling and ecosystem :smiley:.
