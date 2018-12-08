---
title: DynamoDB Pagination Tale Or Expect The Unexpected
date: 2018-12-08 12:49:10
categories:
- AWS
tags:
- dynamodb
- aws
---

{% img /gallery/dynamodb-pagination-tale-or-expect-the-unexpected/header.png 1500 1500 code %}


You have lots of pies — *6 savoury* and *8 sweet*. You take all the sweet pies you can get from the query above. How many do you have?

What happens when you try to filter data in DynamoDB?

Can you guess the answer already?

<!-- more -->

# Intro #

Looking at the query the most obvious answer comes to mind. *It should be 2 pies*. Query sets a limit of two so that's what you have.

Well... Might be, might not be. `limit` might not be what you think it is. And the answer lies in how DynamoDB API implements it's interface.

We will take a look on how to make our data queries more *predictable*.

# Data Set #

First of all we need a table and some data in it.

To create a table we need to specify a primary key. It can either consist of a partition key or both a partition key and a sort key. We will go with the latter option:

*id* — As partition key of type number.

*name* — As sort key of type string.

And here they are as as parameters for the `create-table` command:

{% codeblock create-table-pies.json lang:json line_number:false mark:1,3-4 https://github.com/jshaipuka/jblog/blob/master/source/downloads/dynamo-db/create-table-pies.json View on Github %}
{
    "TableName": "pies",
    "KeySchema": [
      { "AttributeName": "id", "KeyType": "HASH" },
      { "AttributeName": "name", "KeyType": "RANGE" }
    ],
    "AttributeDefinitions": [
      { "AttributeName": "id", "AttributeType": "N" },
      { "AttributeName": "name", "AttributeType": "S" }
    ],
    "ProvisionedThroughput": {
      "ReadCapacityUnits": 1,
      "WriteCapacityUnits": 1
    }
}
{% endcodeblock %}

`KeyShema` attribute is our primary key definition, the `AttributeDefinitions` attribute is the types of our composite primary key and the last attribute `ProvisionedThroughput` is related to how much reads and writes DynamoDB will be processing per second. Reads and writes are set to minimal capacities because our dataset is rather small.

Run this command to create the table:

{% codeblock lang:bash line_number:false %}
aws dynamodb create-table --cli-input-json file://create-table-pies.json
{% endcodeblock %}

## Loading data ##

We will batch load sample data as shown below where `pies` is the table name and `PutRequest` is the action we want to perform.

{% colorquote info %}
Notice how the item attributes have to explicitly specify their types in DynamoDB fashion. E.g. <b>id</b> don't just have it's value set directly but with a type attribute: <b>"id": { "N": "1" }</b>.
{% endcolorquote %}

{% codeblock load-data-pies.json lang:json line_number:false mark:1,3-4 https://github.com/jshaipuka/jblog/blob/master/source/downloads/dynamo-db/load-data-pies.json View on Github %}
{
  "pies": [
    {
      "PutRequest": {
        "Item": {
          "id": { "N": "1" },
          "name": { "S": "Bacon and egg pie" },
          "taste": { "S": "savoury" },
          "main_ingredient": { "S": "bacon" }
        }
      }
    },
    {
      "PutRequest": {
        "Item": {
          "id": { "N": "2" },
          "name": { "S": "Scotch pie" },
          "taste": { "S": "savoury" },
          "main_ingredient": { "S": "minced meat" }
        }
      }
    },
    // truncated
  ]
}
{% endcodeblock %}

Execute `batch-write-item` to insert the data:
{% codeblock lang:bash line_number:false %}
aws dynamodb batch-write-item --request-items file://load-data-pies.json
{% endcodeblock %}

Simplified result:

<div style="text-align: center;margin-bottom: 2em">
{% img align-right /gallery/dynamodb-pagination-tale-or-expect-the-unexpected/pies-table.png 500 500 Pies Table %}
</div>

## Let's try filtering! ##

DynamoDB defines two methods for data retrieval: `scan` and `query`. The essential difference between them is that `scan` reads every item in the table and `query` finds item(s) based on primary key values.

However, that does not mean that the results of a `scan` operation can't be further filtered out. DynamoDB provides a `filter-expression` for that matter:

{% codeblock lang:bash line_number:false %}
aws dynamodb scan \
     --table-name pies \
     --filter-expression "taste = :taste" \
     --expression-attribute-values '{":taste":{"S":"sweet"}}'
{% endcodeblock %}

In the `filter-expression` — the first part "taste" is the column name used for filtering and ":taste" is the filter value. The filter value has to be defined in the `expression-attribute-values`.

When we run that command we get the expected results. Just the 8 sweet pies.

However, suppose you are a pie magnate that has a storage of thousands and thousands of pies somewhere. Will you still get the data just the way you might expect?

As your storage needs grow at some point you will face pagination. Pagination can be applied by you manually or you might hit one of the default AWS default limits which is 1MB result set per Scan operation call.

{% colorquote info %}

DynamoDB limits are listed {% link here https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html AWS DynamoDB Limits %}

<br/>
1MB limit can be calculated based on record size.
{% endcolorquote %}

We can also set a pagination limit by ourselves via the `--limit [number]` on the AWS CLI command.

Let's try that out:

{% codeblock lang:bash line_number:false %}
aws dynamodb scan \
     --table-name pies \
     --limit 2 \
     --filter-expression "taste = :taste" \
     --expression-attribute-values '{":taste":{"S":"sweet"}}'
{% endcodeblock %}

It will produce an output like this:

{% codeblock Scan with limit result lang:json line_number:false %}
{
    "Items": [
        {
            "name": {
                "S": "Blueberry pie"
            },
            "id": {
                "N": "9"
            },
            "main_ingredient": {
                "S": "blueberry"
            },
            "taste": {
                "S": "sweet"
            }
        }
    ],
    "Count": 1,
    "ScannedCount": 2,
    "LastEvaluatedKey": {
        "name": {
            "S": "Blueberry pie"
        },
        "id": {
            "N": "9"
        }
    }
}
{% endcodeblock %}

The `Items` section shows us what data DynamoDB has found when executing our query.

*Just 1 item! Not 2*.

If we take a closer look at these two fields we will get an idea of what happened:
{% codeblock lang:json line_number:false %}
    "Count": 1
    "ScannedCount": 2
{% endcodeblock %}

`Count` is how many items were left after filter was applied and that is what we see in the final result.

`ScannedCount`, on the other hand, means how many items were scanned. It's value is 2 just like our `limit` value.

{% colorquote danger %}

DynamoDB applies it's limits first and only THEN filters out the result!

{% endcolorquote %}

That poses a question on how to get the rest of the items if any limit is applied. As you can see above we get `LastEvaluatedKey` which is the key of our next item. We can use that as a starting position for executing our next `scan` request.


## Solving the problem... ##

Manually:

{% codeblock lang:bash line_number:false %}
aws dynamodb scan \
     --table-name pies \
     --limit 2 \
     --filter-expression "taste = :taste" \
     --expression-attribute-values '{":taste":{"S":"sweet"}}' \
     --exclusive-start-key '{"name": {"S": "Blueberry pie"}, "id": {"N": "9"}}'
{% endcodeblock %}

... And repeat until we go through all the items in the database.

Programmatically (NodeJS) it would look like this:

{% codeblock revised-solution.js lang:javascript %}
// Params to pass to DynamoDB scan
const sweetPiesFilter = {
    ExpressionAttributeValues: { ':taste': { S: 'sweet' } },
    FilterExpression: 'taste = :taste',
    TableName: 'pies'
}

async scan(params, resultSet = []) {
    const [Items, LastEvaluatedKey] = await db.scan(params).promise()
    const result = [...resultSet, ...Items]

    // If there are more items to fetch then we should do that
    if (LastEvaluatedKey) {
        const updatedParams = Object.assign({}, params, { ExclusiveStartKey: LastEvaluatedKey })
        await read(updatedParams, result);
    }
    return result
}

const allSweetPies = await scan(sweetPiesFilter) // 8

const onlyFiveSweetPiesFilter = Object.assign({}, sweetPiesFilter, { Limit: 5 })
const onlyFiveSweetPies = await scan(onlyFiveSweetPiesFilter) // Still 8!
{% endcodeblock %}

However, there's a problem with this script.

Suppose we actually need to limit our data to a certain pagination limit precisely. Say 10.

Loop until you get 10 items if they are present and then break out of the loop.

## Revised solution ##

{% codeblock revised-solution.js lang:javascript %}
// Params to pass to DynamoDB scan
const sweetPiesFilter = {
    ExpressionAttributeValues: { ':taste': { S: 'sweet' } },
    FilterExpression: 'taste = :taste',
    TableName: 'pies'
}

async scan(params, enforcedLimit, resultSet = []) {
    const [Items, LastEvaluatedKey] = await db.scan(params).promise()
    const result = [...resultSet, ...Items]

    // Determine if we need to fetch more items 
    const noEnforcedLimit = !enforcedLimit
    const enforcedLimitNotReached = enforcedLimit && result.length < enforcedLimit
    const shouldGetMoreItems = LastEvaluatedKey && noEnforcedLimit && enforcedLimitNotReached

    if (shouldGetMoreItems) {
        const updatedParams = Object.assign({}, params, { ExclusiveStartKey: scanResult.LastEvaluatedKey })
        await read(updatedParams, [...resultSet, ...scanResult.Items]);
    }
    
    // Discard items if there are more than we want
    return enforcedLimit ? result.slice(0, enforcedLimit) : result
}

const onlyFiveSweetPies = await scan(sweetPiesFilter, 5) // 5
const onlyTenSweetPies = await scan(sweetPiesFilter, 10) // 8. That's all there is
{% endcodeblock %}

Bingo!

## Takeaways ##

Be careful about your assumptions on command's parameter names and read the official documentation carefully.

Also, try out the code above :)
