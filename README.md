# DynamoDB Patterns
A set of patterns for using DynamoDB, sometimes with AWS Lambda functions

## Introduction
DynamoDB is a serverless key-value store database from AWS. Its a very useful tool to get up and running quickly with an idea
and may be the ideal database for scaling in certain situations. 

The data model is quite limited. As a primary key you can have either a hash key, or a hash key along with a sort key.
For a give hash/sort key you can have a set of arbitrary attributes (a la a document), which can be of map, set or list type as well as the usual 
String, Number, Boolean types.  

You can either query on the hash/sort key, where the hash key is an equals match, and the sort key allows range queries. 
The alterative is a scan operation which will scan the entire table (although it is paginated). To do any sort of complicated 
querying you can filter on the results of a scan result (or query result for that matter), bit this will be after a full scan has been done. 
The implications here are that it will cost you as much and it will take as long as scan without filter. Also, the max pagination and payload 
sizes apply to the pre-filter scan results. Hence, it's useless for finding a needle in the haystack and probably not good for any sort 
of live use-case. 

A very useful feature of Dyanamo is that all inserts, updates and deletes are written to a kinesis-like stream that persists for 24 hours. 
Hence the Dynamo table can act as a sort of persistent event log. 

You can also set the time-to-live on a dynamo entry. More recent additions include global tables and transactions. And that's about it!

Most of the patterns presented below try to use this restricted data model to our advantage. 

## The patterns 

1. Aggregator: Merge and collect records under a hash key and use a lambda function to decide when it is complete. 
2. Periodic aggregator: Collect data under a given hash key and use a cloudwatch event triggered lambda to process them 
3. Processor: Use dynamo to store state of ongoing transactions.
4. Optimistic Locking: Avoid race conditions by using a version attribute and conditional writes
5. Historic: Stream all writes to another append only dynamo table

### Aggregator

**Motivation**: Imagine you have a stream of user events, and you would like to trigger some processing on those events
when the collection is a certain size or contains certain data. 

**Examples**: 
1. Lets say you collect clicks on your webpage. Clicks are loaded into kinesis, a lambda processes the events and 
adds the events to a list on a dynamo entry where the hash key is the session. As the user clicks the list grows and the lambda is triggered on each insert. 
The business guys want a notification each time someone does more than 10 clicks on the page. 
Our lambda is triggered by the dynamo stream, and emails out any time the list is greater than 10. 
To avoid emails on 11, 12 etc, you would like to store the state somewhere, and why not on the dynamo table itself? Although, try to avoid an infinite loop!
2. Suppose you have 2 data streams that you need to merge before processing. Suppose you collect recommendations for 
a user but you only want to email them after they are active so as not to annoy them. You could merge the recommendations and login events and process when they're both present. 

**Code**: 

**Pros**: 
1. Data is processed as soon as its available. 
2. You can set multiple triggers on a given stream to have different functionality for given 
collections of events (although this gets complex quick, don't use is for fraud detection!)

**Cons**: 
1. There are a lot of wasteful lambda invocations. If we need 10 events to continue, we will have 9 wasted lambda invocations. 
Until Dynamo introduces some sort of conditional streaming logic there is no way around that. 
2. Although we have the data persisted in dynamo, it is very hard to actually query the table for unmatched/incomplete sets because
of the limitations with scan. Perhaps you could get around this with an index. 

### Periodic Aggregator
