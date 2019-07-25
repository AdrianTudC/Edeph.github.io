---
layout: post
title:  "Trying out DynamoDB"
date:   2019-07-24 09:04:02 +0200
categories: libs
---
This week at work I had to move some persistent data from browser's localstorage to a database in order to make some user preferences available cross-device.

Naturally the first question that popped up was: what database to use? Nosql was pretty much a no-brainer since I didn't have any complicated data to keep, but what flavour of nosql? Me personally, I'm a big fan of mongoDB, I've been using it for 5 years now and I think it matured well, I would have also finished the implementation faster because I know the API (almost) inside-out. But due to some deployment-related restrictions I decided to give DynamoDB a try. After all, why not add another tool on my belt.

But oh boy I was in for a surprise.

## The lib structure

I spent the first 30 minutes looking around and I found 2 kinds of code examples/documentations which confused the hell out of me. Both documentations were using the `AWS-SDK` npm package but one was using a lower-level API and the other one was instantiationg a `documentClient` object which was used as a handler. I also bumped into another deprecated git [repo](https://github.com/awslabs/dynamodb-document-js-sdk) but I just left that alone since I already had a handful of choices (actually two :v:).

The higher level API:
{% highlight javascript %}
import * as AWS from 'aws-sdk';
AWS.config.update({
  region: '', accessKeyId: '', 'secretAccessKey': ''
});
const docClient = new AWS.DynamoDB.DocumentClient();
{% endhighlight %}

And the lower level:
{% highlight javascript %}
import * as AWS from 'aws-sdk';
AWS.config.update({
  region: '', accessKeyId: '', 'secretAccessKey': ''
});
const docClient = new AWS.DynamoDB();
{% endhighlight %}

The difference is subtle but it affects how you are using the API. And to be honest I didn't find the docs to be very upfront mentioning this difference.

In the end I stuck with the lower level API because I prefer barebone stuff and because i'm a control freak :grimacing:;

## CRUD

With the initialization out of the way, I thought that it's gonna be easy peasy to just create some data, make a getter for it and an update function. But again, I was wrong and it took me a while before I got to understand how the querying works.

Another thing that is not really emphasized (especially when looking up code examples) is that the lower level API not only has different methods than the `DocumentClient` API, but requires you to have a different structure in your query. You have to specify yourself each data type that your fields will contain, which I find completely weird and unnecessary for a nosql database but I did my best to see past my personal beliefs and went on with it.

Let's see an example of a getter:
{% highlight javascript %}
const params = {
  TableName : 'Table',
  Key: {
    someKey: 'pretty cool string'
  }
};
// this gives back the entire document that has the primary key with the value 'pretty cool string'
{% endhighlight %}

This looks decent, nothing out of the ordinary.

Onward to inserting a value:
{% highlight javascript %}
const params = {
  TableName : 'Table',
  Key : {
    someKEy: 'pretty cool string'
  },
  // so far the selector part is the same as above
  ConditionExpression: 'ADD #COLUMN :VALUE',
  ExpressionAttributeNames: {
    'COLUMN' : 'cool_column'
  },
  ExpressionAttributeValues: {
    'VALUE' : {
      S: 'some crazy value'
    }
  }
}

// this adds a string item to a string set
{% endhighlight %}

Woah! This seems... too much, but for longer queries it can be useful to avoid huge one liners (altough I prefer mongo's json-style querying rather than this verbose one). To be honest if you don't want to use `ExpressionAttributeNames` and `ExpressionAttributeValues`,  `ADD mycolumnname ${myvalue}` does work (for non-list data types). The docs show this expanded way of writing it as being the default (in fact, the docs definition looks [scary and intimidating](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#putItem-property)).

## Not the best not the worst

Of course it's not all a mumbo jumbo and everything makes sense in a funky way. You can group primitive types using lists (type `L`) and it would look like this in a table: `[ {S: 'some string'}, {N: 42} ]` which is kinda weird. There is another type called Set that looks like your average array when you look at it: `['some string', 'another boring string']` but this is actually a set and it has the type defined as `SS` if it's a set of strings or `SN` for a set of numbers and so on...

These two types also use different syntax for operations. Lists use the `SET` keyword (ironically) while sets use the `ADD` and `REMOVE` keywords. You can use the `REMOVE` keyword for lists also, but you need to know what index the item has in your list, and I believe in most cases that implies that you need to query beforehand and count yourself which seems tedious. There are also a bunch of helper functions that you can find in the docs like `list_append(<list>, <item>)` which you use to add items to a list.

In the end I'd say that for super basic stuff DynamoDB suffices and it's pretty easy to get it up and running (provided you bump into the proper docs and examples) but for a more ample project I would still use mongoDB :robot:.