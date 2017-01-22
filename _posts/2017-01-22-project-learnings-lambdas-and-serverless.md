---
layout: post
title: Project learnings using AWS's Lambda with the Serverless Framework
---

We used the Serverless Framework and the Serverless Webpack plugin to write our lambda code in ES6. Our journey came with a few learnings.

# Weigh up costs of wanting to code in ES6

When we used Serverless Webpack with Babel, we wrote cleaner and more concise code using ES6. We had all
the goodies to work with such as classes, let, arrow functions etc. However, we paid premium taxes for the setup fee.

We had to include a plugin 'Serverless Webpack' to enable ES6.

Serverless Webpack was great in many ways. We had the command `serverless deploy` to add to our
deploy script which automatically bundles our code. We also had to option to deploy multiple lambdas in our project at once.

Deploying multiple lambdas was achieved by defining an entry point in the `webpack.config.js` file for
each respective lambda:

```javascript
module.exports = {
  entry: {
    'future-article-processor': './src/future-article-processor/handler.js',
    'newscorp-message-reader': './src/newscorp-message-reader/handler.js',
    'newscorp-message-worker': './src/newscorp-message-worker/handler.js',
  }

  /// More
};
```

There were two main drawbacks:

    individual functions cannot be deployed without AWS cloudformation being updated; and
    reading your code on AWS’s inline code editor may require cataract surgery.

Uploaded code that has been transpiled from webpack will retain its form:

![AWS Inline Code Editor](https://raw.githubusercontent.com/wsquared/jekyll-now/master/images/2017-01-08%20AWS%20inline%20code%20editor%20with%20webpack.png)

We had to run the command `serverless deploy`. This redeployed all of our functions and the process of updating the cloudformation template also occurred. The combined effects increased our feedback loop and slowed our abilities to develop faster.

There are [discussions about individual function deployment using Serverless webpack](https://github.com/elastic-coders/serverless-webpack/issues/60) and may be available in the future.

In the interim, if you can withstand not having the ability to deploy individual functions quickly such
`serverless deploy function -f functionName`, then Serverless Webpack is the gateway to ES6 goodness.

# Beware of the rogue lambda - clean up after yourself

As we refactored our lambda code, we decided to rename our lambda. Using the Serverless Framework, we renamed our functions in the `serverless.yml`:

```yaml
// More code above this line
functions:
  futureArticleProcessor: // Can be renamed
    handler: src/future-article-processor/handler.futureArticleProcessor // Can be renamed
    events:
    - schedule:
        rate: rate(1 minute)
        enabled: true

```

When we redeployed our lambda, a new lambda was created. Unbeknown to us, there were two lambdas that did the same thing, but with different names.


Both were listening to messages from the same SQS queue. The old lambda became known to us as the 'rogue' lambda.

How did this 'rogue' lambda get its name?

We develop our lambda to ignore a message that was future dated. When the message was in the past, the lambda moved the message from one SQS queue to another. The lambda was scheduled to receive messsages from the SQS queue every minute.

![Future Dated Poller](https://raw.githubusercontent.com/wsquared/jekyll-now/master/images/2017-01-22%20Future%20dated%20poller.png)

How did this cause an issue?

We had unpredictable behaviour because there were now two lambdas receiving messages from the same queue. Although scheduled to run every minute, they started at different times.

We spent hours looking through our code and Cloudwatch logs. The mysterious bug that kept eluding us was the 'rogue' lambda not because of a bug in our code, but missing what was infront of us.

Make sure you clean up your renamed lambda.
