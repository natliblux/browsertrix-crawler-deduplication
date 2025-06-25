# Introduction
This is a fork of Browsertrix Crawler, with an added cross-crawl deduplication feature. If activated, it will use a Redis database to store page hashes and look them up during crawl time to deduplicate already visited pages.

Since many pages can have dynamically changing code (such as timestamps, variables, ...), these elements can distort the computed hash in an undesired manner. For this reason, an additional detection algorithm is included in the crawler in order to make the deduplication as effective as possible.

This feature makes it possible to remove dynamically changing elements on the page from the hash computation by looking up a set of rules, defined by the user and stored in the same Redis database. At runtime, these are used by the crawler to remove the offending elements from the page before calculating its hash. Note that the page itself remains unchanged, these removals only affect the hashing process.

A complete example is given below.

# Requirements

The same requirements as the original Browsertrix Crawler, plus a Redis instance for storing deduplication data.

# Build

Clone the code and run:

    docker-compose build

# Usage

Run the crawler as usual, and add the following parameters:

  - `crossCrawlDeduplication`: boolean, default **false**,
  - `crossCrawlDeduplicationRedisUrl`: string, URL of the Redis instance, prefixed with `redis://` and including the port. An example is: `redis://my.redis.url:6379`.

Here is a full command-line example that runs the crawler using Docker, with cross-crawl deduplication enabled, and a Redis instance deployed on `my.redis.url`on port `6379`:

    docker run webrecorder/browsertrix-crawler crawl --url "https://webrecorder.net/" --generateCDX --collection "test" --workers 2 --crossCrawlDeduplication true --crossCrawlDeduplicationRedisUrl "redis://my.redis.url:6379"


## Output

At the end of the crawl, Browsertrix outputs some crawl statistics in JSON format. Among these statistics, you can now also find the total amount of deduplicated pages:

````
"message":"Crawl statistics",
"details":{
	"crawled":1,
	"total":1,
	"pending":0,
	"failed":0,
	"limit":{"max":0,"hit":false},
	"pendingPages":[],
	"dedupedPages":1
}
````

Note the added statistic `dedupedPages` at the end of the JSON.

# Setting up deduplication rules

As mentioned before, you can define rules to remove elements on a page from the deduplication hash process. These are in the form of regular expressions that match specific elements on the page, and are stored in a Redis database in the form of (key,value) pairs. Each rule is identified using the key:

    dedup-regex-pattern:<id>

where `id` is simply a name you can give to the rule. You can define as many rules as you want.

The value corresponding to each key is then the regular expression you wish to use. We will illustrate this with a small example.

# Example

Consider the HTML snippet:

````
<html>
<head>...</head>
<body>
	Some text.
	<p>The date is: Wed, 25 Jun 2025 06:26:48 GMT</p>
</body>
<html>
````

Since the paragraph `<p>The date is: ...</p>` contains an automatically generated timestamp, it would make no sense to try hashing this page as it would be different each time. Therefore, we need to remove it before hashing.

For this, we set up the regular expression `'<p>The date is .*<\/p>'`, which will match to the paragraph containing the date, and this will tell the crawler to remove this snippet before computing the hash. You can add this regular expression to the Redis database on the command-line as follows:

    redis-cli SET "dedup-regex-pattern:date-paragraph" "<p>The date is .*<\/p>"

Now when you run the crawler, it will load this regex:

    Loaded regex: dedup-regex-pattern:date-paragraph - <p>The date is .*<\/p>
    Successfully loaded 1 deduplication regex pattern

When it encounters the page above, it will remove the timestamp paragraph before hashing it, because it matches the regex defined previously. Thus, the part of the page that will be visible to the hashing process will be:

````
<html>
<head>...</head>
<body>
	Some text.
</body>
<html>
````

Once the crawl is finished, you will see that your Redis database contains the deduplication hash for this URL:

    redis-cli GET "http://my.test.url"

Thus, each time you run the crawler, it will compare the hash computed at runtime with the hash in the database. 

This mechanism enables deduplicating pages across independent crawls.

