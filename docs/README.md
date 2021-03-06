# How it works

This application architecture is made up a few pieces:

### Discovery

![discovery](/docs/discovery.png)

For discovery the application makes use of both a polling approach (used for PyPI and RubyGems) and a
subscription approach. You can find the code for [polling RubyGems](/app/lib/ruby-gems.js) and [polling PyPI](/app/lib/pypi.js).
This code is deployed to run as two Lambda functions, each one executing once every 5 mins.

NPM is a much more active registry with a lot more packages being published so rather than resorting to
a less efficient polling method the application uses the public CouchDB interface of NPM. It [implements
a CouchDB follower](/app/npm-follower/watch-npm.js) which gets a push from NPM whenever a package is updated.
This code is deployed as a docker container that runs in AWS Fargate.

All discovered Github repos from all sources get published to a SNS topic.

### Crawling

![crawling](/docs/crawl.png)

The SNS topic is the entry point to crawling, receiving pushes from all discovery sources.
On publish to the SNS topic a lambda function is triggered which crawls the repository, generates an
HTML and JSON artifact for any discovered changelogs, saves metadata into a DynamoDB table,
and finally publishes a notification via [Redis Pub/Sub](https://redis.io/topics/pubsub).

### Push to browser

![push to browser](/docs/push.png)

The Redis Pub/Sub is monitored by a container running in Fargate that uses [socket.io](socket.io) to
keep a persistant socket connection open between the browser and the backend. It pushes a notification
to the browser in realtime whenever a repo has been crawled, which is used to make the live updating
feed on the homepage. There is an Application Load Balancer providing a gateway to connecting to an
instance of the socket.io container that will push realtime notifications to the browser.

### Autocomplete Search

![autocompleted](/docs/search.png)

The search field on the homepage is backed by an API Gateway with a lambda function behind it. The
lambda function just queries the search index that was generated by the crawler, and returns the
results to the homepage.

### End to end

![end to end](/docs/architecture.png)

At the top level of the stack there is a CloudFront distribution which glues all the various resources
together into one coherent website. The static content gets fetched from S3, while search queries are sent
to API Gateway, and socket connections get sent to the docker container via the Application Load Balancer.