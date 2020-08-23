# article-feed-system-design
High level approach on how to design a twitter like system

## Functional requirements for the system.
- End User can find and follow a professional profile
- A Professional can upload and post an Image with text.
- Uploaded post will appear in professional’s feed as well as users following the professional.
- Post should be appear in followers feed immediately if user refresh feed page or auto-update at a frequent schedule.
- Professional uploaded feed also appear on own profile. Able to update/delete.
- Other users can like and comment on feed as well as report.

## Design Considerations/ Assumptions to make the context bounded.
We only allow verified profiles/professionals to publish
- Number of active users - 10 million
- Number of professionals - 5k
- Number of posts per day(2 for person) - 10k
- Size of the image - 5 Mb Max (1500 * 500)
- on average each professional has 1 million followers
- on average each user follows 1k professionals
- read through put - say 20mn per day
   
## Considering  Push Vs Pull
Unlike twitter, we have a small number of professionals who publish their work, so for publishing we will use push.
we need to avoid the load on the server, since we will have the control on the client apps, we can enforce client to do the push and pull.
A professional can have millions of followers, so pushing the post to all these clients could be expensive on the server side,
So we have to make the client to poll/pull at fixed intervals or when client refreshes. Of course we need to pre-compute their timelines and cache them otherwise it will be hard to deliver the that read through put.

- Professionals - push their articles, this fan-out is the expensive request<br>
- Users - pull for their feed (active users(logged in last month) will have a cache of the timeline with 500 posts/entries).
- design will be read optimised, writes will take time based on number of followers.

## Making the technical choices.
To make it more fun, let's design the system in a cloud environment like AWS, so I will try to make these choices in cloud. 
Streaming date at scale requires tools like Kafka, AWS recently announced MSK (managed streaming for Kafka in AWS), its similar to Kinesis streams in AWS.
Point is doing things in cloud gives you more options in some areas and limits in some other. if you were to do this before MSK we will probably choose Kinesis,
which is different than Kafka but does the job.

### Basics (How we allow traffic into our resources)
We first need the basic network configuration in the cloud.
As Zeff Bezos insists systems should talk with APIs only, so we basically will design APIs to serve this functionality.
Let's Just assume that we have domain name with which we want to expose our APIs.


- VPC to isolate our resources
- our domain name api.mydomain.com (We can't dynamically change the endpoint based on the hosted zone, so we need a domain)
- Route53 - this will translate DNS endpoints to actual IPs also helps to register, but it also allows us to configure aliases for public/private hosted zone. we will configure our custom domain here to route the traffic to our API gateway.
- API gateway - this is where we configure our APIs and define endpoints it will also authorizes the clients in coordination with cognito.
- Subnet we at least need one subnet as we are going to launch Elastic Service and other resources..

Lets design part by part..

### Identity and Access management(How clients access the APIs)
- Amazon Cognito Federated Identities
- Amazon Cognito User Pools
- Amazon Cognito Sync
The first two suit best to us, first one is when we want users to use other identity providers, second one is when we want authorise using the user pools in Cognito.

Since we are using AWS, Cognito is our best option to control access to our resources, it allows us to use protocol we want to use for securing the resources.
The application needs to be mobile centric, and most of the apps like twitter, just keep the user logged on using shared preferences on client side. we should allow users to login with other providers like google facebook etc...

 ![alt text](images/api-access-control-flow.png?raw=true)
 
 We will expose the APIs to register and sign as well through this flow. we can extend this to use other IDPs to make it easy for users. 
 Roles will only have specific access to invoke particular methods on the endpoint. there are so many ways to achieve this. In most/all the cases we will have Idtoken(claims of the user), access token (claims for the resources), refresh token (to refresh an expired token), 
 the ways of designing this is basically, how we make sure that the token will have only required claims because if they have more claims they can assume the role mapped and create modify the resources. 

#### Documents in all the data stores of a user will be linked by user_id, index sharding will also be based on user_id, we can use something like Snowflake by twitter for even sharding of stores.

#### User/Professional Profiles

##### User creating a profile
- a record will be created in Cognito which will take care of user authentication and his identity claims.
- A record wll be create in dynamoDB which will create a record in elastic.

##### Profile-Photo
An uploaded photo is static, so we will deliver the images using CDN(cloudfront), backend for the CDN will be S3,
So when user is uploading an image, we first make the call to upload it in the S3, we don't need the random prefixes for bucket names anymore to make it fast, we can store however we want and performance won't be impacted.
We can create the buckets on the user names for their profile picture as well as the images they post when publishing an article.
- /api/vi/image [POST] will return a CDN link, which will be referenced in all the documents which have this image.
- flow will be api gate way --> lambda-->s3, lambda generates the cdn link based on the domain and bucker name.

##### Profile Storage and Search
Cognito has the basic infromation already, it will take care of login information and everything about the security.

Our next task is, users will look up for professionals and/or other users, and we need to give the search results really fast.
Of course we paginate the results, the first query will give the 10 matching results and if the user needs to more, he can me another request, front end will take care of this.

Lets look at the possible data stores we can use. we can use sql databases, no matter how much we shard/partition, it will be O(log(N)) to locate the user, N- being the number of records in the partition.
Search is another issue, we need a number of results that match the query, if search on the pattern in sql, there is no way to locate the partition of the user so it will be O(N) and incredibly slow.
So yes we can't use sql, lets look at the noSQL options in AWS

Our document structure will be something like 
```
{
        "username" : "hernandez94",
        "firstName" : "Jennifer",
        "middleName" : "Maria",
        "lastName" : "Hernandez",
        "image-url" : "https://cdn.mydomain.com/user-id/profile.photo",
        "followers" : "0",
        "following" : "12",
        "mobile" : "919834739847",
        "email" : "personal@email.com",
        "addresses" : [
                 { "type" : "home", "addr1" : "1929 Crisanto Ave", "address" : "Apt 123", "addr3" : "c/o  J. Hernandez", "city" : "Mountain View", "state" : "CA", "country" : "USA", "pcode" : "94040" },
                 { "type" : "work", "addr1" : "2700 W El Camino Real", "addr2" : "Suite #123", "city" : "Mountain View", "state" : "CA", "country" : "USA", "pcode" : "94040" },
                 { "type" : "billing", "addr1" : "P.O. Box 123456", "city" : "Mountain View", "state" : "CA", "country" : "USA", "pcode" : "94041" }
        ],
        "createdate" : “2016-08-01 15:03:40”,
        "lastlogin": "2016-08-01 17:03:40",
        "loc": "IP or fqdn",
        "roles" : [professional,user],
        "doc-type" : "user"
}
```
DynamoDB - yes a good candidate with millisecond latency to locate a document, if we use username hash to store the document and if we have the exact username then yeas it will be really fast.
But people are dumb they may not type exact name or may stop at first name only or the last name, other than this DynamoDB will not give us full search capabilities.
But dynamo has concept of global tables, which replicates data in all the regions and reduces the latency on the basis of user location.
Our issue is how do we search with just a pattern for the matching profiles ????

Yes you guessed it right, Lucene or ElasticSearch which is built on lucene can give us that full text search capability. but if you have millions of documents indexing becomes slow.
and that is not negotiable if we understand the internals of Lucene. Also disaster recovery in ES is on the basis of snapshots you take of the data at a frequency, generally people don't like this.

So how do we make the profile search faster ? shall we mix these two ? or just use elastic search alone ?
We are not twitter, we only have hand full of professionals at least for now, searching in 5k profiles is not an issue, even DynamoDB alone can give the results pretty fast.
But if this professional base grows then it becomes slow, but to be robust lets consider, we want any user to appear in the search.

So yes we have to mix these two to make the profile search faster, unless we are not afraid of using elastic as primary data source. Also we need shard the date into different indexes,
because if the index is big then the lambda we use to sync the data might timeout. So we shard the profiles into different indexes so it will be fast.

Below is how we keep these two things in sync, dynamo hashing will be done on the basis of the name, also the elastic documents will be indexed on the name fields, though we can index other fields too.
we can dig deep into how do we shard the indexes and etc..
 ![alt text](images/dynamo-elastic.png?raw=true)

##### So who is following who ? how to store that graph ?
Again we can simply store in sql but it doesnt scale, so as the relations suggest, if we want to store this data without duplicates then it forms a graph structure.
then we can do Dijkstra's algorithm to find out, second and third level connections like linkedIn :), still waiting for a day where I can write Dijkstra's algorithm in production code :D

The Obvious answer is Amazon Neptune - a Graph database which can store billions of relationships.
It is incredible because,
- vertices can have labels and properties
- edges can have labels and properties
- vertices can be of different type
- edges can be of different type
great !! so why can't we just store all the user data here? and their posts as well, also the their relationships, and comments, likes , reports...
hold on, technically yes we can store all that here, and we can query as well, but how do you do full text search on users or posts ? well we can't.

we will be storing user profiles and the posts in elastic search so we can support the text search.
what we can store in neptune is 
- if user is following 1000 professionals, then that user vertex will have 1000 edges each pointing to the vertex of the professional he is following, later when we want to pre-compute his timeline we will use this.
- we can store the posts of the each professional as well but they are not sorted and we will be storing the posts in elastic any way, so we have no use case for this
- if user wants to save the posts he liked then we can store those IDs of the liked posts for the vertex of the user.
- actually we can store, comments, likes, for all the users in this graph as well, but we won't compute the time line of the user based on these, we compute them based on the prrofessionals he is following, or may be we can use this as well?

For now we just store the the follower/ee relation ship here, lets see later if we need to store anything more here.

Neptune comes with its own complexities as its not a traditional database, we need tools to ingest data and read it.
so we need to use something like SparkQL or Gremlin to deal with it, we have to run these clients somewhere in the cloud, so it could become expensive. If cost is not a constraint I would use this.

So when do we actually need this relationship ? 

When we are computing the timeline/feed for a user, we need to look up who he is following and pick their newest posts and make a feed of say 500 entries.
The algorithm to compute the posts can be more complicated. Our Idea is that for the recent users (who logged in past 30 days) we already compute their feed with 500 posts and keep it in the redis cache.
Intention is we will server the feed from cache, otherwise we will never be able to meet the read throughput required.

Also when a professional posts, we want to know who are following him, and ingest that post in their feed cache. the write/fan out is going to be slow anyway, the read needs to be fast, we can argue that we will use sql database and shard it into 100s of partitions to make the query faster.
that will be cheap, also we don't have enough use cases for using neptune, as we are using elastic for most of the tasks.

If there is no ItemSize limit we can even use DynamoDB for this purpose.

So the suggestion will be to use sql, till the user base grows really large and its not possible to scale any more, then we can use this data and create the relationships in neptune, and handle that problem later.
we will have a user table and followers table like.,

believe that your schema requires a union table to assemble the information you need; and it may be more efficient to do this in multiple tables. To maintain a separate table of followers with (possible) duplicate information from users may also be undesireable. A more efficient schema would be:

users tables-

| uid | name      | 
---|:---|
|   1 | Phillip    | 
|   2 | Another    | 
|   3 | Some Crazy |
|   4 | Nameless   | 

followers table-

| user_id | follow_id |
---|:---
|       1 |         4 |
|       2 |         3 |
|       3 |         2 |
|       4 |         2 |


Isn't his join is going to be slow ? Yes it will be but our goal is not use this join, we will be read efficient with caches, and avoid computing this.

But these relationships can also be cached, Memecached is really good at sharding data, so we will use this for caching these relations, but we need to decide whose relations are we going to store, they have to be professionals who are really active and posting frequently,
we could need more data to decide this, if the user base is small we just cache all the relations.

### Post/Write APIs 

#### A professional is posting with an image and text.
- image will uploaded and we will get a cdn link to be used in all documents referring the image.
- this post should go to elastic search and get indexed.
- professional's feed cache in redis needs to be updated, only update the document id from elastic.
- feed cache of the users who are following him needs to be updated, only the document id.

First we need to get the document indexed before inserting in required places, and it could take time.
we can have heavy load of posts or if we do this in the single call our lambda can be timed out, so we need an asynchronous mechanism here, like queueing or streaming.

We will use Kinesis for even streaming as we need data ingestion to be fast, first lambda subscribes to lets say 'posts' topic, it reads each post and indexes/sends to elastic cluster, once the document is indexed it publishes back to another topic 'indexed-topic' with document id.

there will be two lambdas listening to 'indexed-posts', one writes to professional feed cache, one writes to feed caches of all the users following the professional (hopefully 15 min is enough or else we need look for other way).

- now the post is ready, the next time user refreshes/calls the feed api or app polls for new feed, we deliver it from the redis cache, same will happen for professional's time line as well.
- we will only store the document_ids in the redis cache, till we grow the users we can actually store the entire feed for the user in the cache with 500 entries, other wise we can parallel query elastic for those documents and make the feed for the user.

#### user follows/un follows a professional
Obviously we get both user_id of user and the professional
- we need to update the followers table/neptune if we are using it
- get the professional's posts and insert in user's feed cache
- update dynamoDb, update follower's count on the profiles

Same concept as above first lambda publishes the message to a topic say 'followed', there will be three lambda's subscribed to it, one will update the tables ,  one will update the cache, and one will update dynamoDB which will sync to elastic.

when user refreshes/app polls automatically newly followed professional's posts appear in the user feed as we updated the cache.

#### User Comments/Likes/reports a post
I want to delegate this also to elastic search, because it will allow us to search on the basis of comments as well, 
Elastic search handles nested documents well, and we will be commenting on already indexed document so, it should be fast as well. 

Only problem will be document getting large because of comments, but remember you will be searching for a unique document in elastic, it will be superfase, because lucene will store the nested documents also in the same block as parent.
We can work on the query part to deliver the paginated comments to the original document.

A simple nested document may look like - same for comments/likes/reports
```
"mappings": {
    "series": {
      "properties": {
        "comments/like/report": {
          "type": "nested",
          "properties": {
            "name":    { "type": "string"  },
            "user_id": { "type": "int"  },
            "text/like/report":   { "type": "short"  },
            "date":    { "type": "date"    }
          }
        }
      }
    }
  }
```
#### Failures/fall back
Our's will be a eventually consistent system because of heavy fan out requests, but it is possible that our lambda functions could timeout or fail to execute.
So we will define a dead letter queue mechanism after 3 attempts to process the data, this can be a ptoblem with kinesis as it won't allow the next records if one lambda is stuck at a record, not sure if that behaviour is changed now.

DLQ- will trigger a notification email/message.

#### ElasticSearch Service
This in itself is huge problems as we need to decide on the indexes, partitioning, instances, backup etc.. we need a lot of data to decide these things, and there is no accurate answer, its our of scope for this article.
 
 #### the IAM roles
 there number policies the needs to be created for these resources, then these policies needs to assigned to corresponding roles. these are the roles users/apps going to assume when they have valid token.
 ### final picture
  ![alt text](images/architecture-feed-app.jpg?raw=true)
  
  Diagram explanation
  1. Cognito authentication, app must get the cognito token first and then exchange it for resource/API access.
  2. [POST] - api/v1/image - when we upload post/profile picture, we first update in s3 and get the cdn link.
  3. [GET] - api/v1/image?id=xxx - retrieve the static files through CDN.
  4. [GET] - api/v1/feed?user=yyy&quantity=20&start=100 - must mention how many results should return in response, this will be our core API, this is what is used to pull the feed for a user, this is what app calls in the backgroud to refresh the feed. this will be pre-computed for the active users, when a professional posts this will be updated with the recent post, we will store a standard 500 entries if users scroll more than that, then we need to fetch the old posts.
  we will first check in the memcached first if the relation is available, if not we will look into the relational DB, for the relationships, and then fetch their old posts and deliver in api in paginated manner.
  5. [POST] - api/v1/post - as explained above we will insert this in the professional's and followers' feed in redis, so first we look up in memcached for relations then in sql.
  6. [POST] - api/v1/user - we create the user profiles in dynamoDB, initialising their follwers/followees to zero, this will sync to elastic for search enabling.
  7. [GET] - api/v1/post?id=xxx or q=text [GET] - api/v1/user - to retrieve a matching document or user or all the matching documents. queries are directly done on elastic.
 [POST] - api/v1/comment/like/report - this will directly update the document in elastic.  
  8. [POST] - api/v1/follow - [POST] - api/v1/post creating a post/when followed, there are multiple actions need to be performed, so we first publish to kinesis streams, and then data will go through different states till it updates required systems.
  9. Can be same flow for 8 - different lambdas perform different actions on the data streams as explained above.
  10. as in 5 when post happen's one of the lambda's insert the post in the feed's of the users and the professional.
  11. like 8 when professionals gets followed we update the record in dynamoDB
  12. All the resources like lambda etc will log the messages in cloudwatch, we will stream those logs into elastic search cluster, so we can search using kibana.
  13. Log streaming to ES cluster
  14. using Kibana for log search 
  15. When a post happens we need to index the document first, when someone comments/likes/reports we will directly update the document with the comment/like/report. 
  When constructing the feed timeline also we will query for the docuemnts of a professional, basically all the search queries happen on elastic.
  16. Fetching the relationship graph if its not available in memcached, as well as updating them when user follows/un follows.
  17. DynamoDB streams will continuously update the user profiles in ES cluster.
  18. A rare case when the feed of 500 entries is not enough and user requested more posts, then it will check in the sql, for the graph after checking in memcached.
  19. same as 18.
  
  #### Extras
  - this is a high level design, we really need to dig deeper and need more data to make this concrete.
  - I have referred the documents form aws portal and twitter developer portal.
