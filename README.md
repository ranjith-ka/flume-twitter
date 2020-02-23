Analyzing Twitter Data in CDH/HDP
==================================

Configure Twitter Application
------------------------------

 Go to [Twitter Apps](https://apps.twitter.com/) and click Create New App. Fill in required information to create a new app. Go to the Keys and Access Tokens tab and save securely following keys:
<pre>
 Consumer Key (API Key)
 Consumer Secret (API Secret)
 Access Token
 Access Token Secret
</pre>

 Above Keys will be used in Flume application to capture the Twitter data.

Configure Flume
----------------

1. **Build or Download the custom Flume Source**

   A pre-built version of the custom Flume Source is available [here](http://files.cloudera.com/samples/flume-sources-1.0-SNAPSHOT.jar).

   The `flume-sources` directory contains a Maven project with a custom Flume source designed to connect to the Twitter Streaming API and ingest tweets in a raw JSON format into HDFS.

   To build the flume-sources JAR, from the root of the git repository:

   <pre>
   $ cd flume-sources
   $ mvn package
   $ cd ..
   </pre>

   This will generate a file called `flume-sources-1.0-SNAPSHOT.jar` in the `target` directory.

2. **Prepare the HDFS location for Flume data**

	<pre>
	$ mkdir /Flume
	$ hdfs dfs –mkdir –p /user/Flume/tweet (location will be used by Flume to capture the data)
	</pre>

3. **Create a Flume configuration file in local directory**

	Example configuration file given below.

	<pre>
	TwitterAgent.sources = Twitter
	TwitterAgent.channels = MemChannel
	TwitterAgent.sinks = HDFS

	TwitterAgent.sources.Twitter.type = com.cloudera.flume.source.TwitterSource
	TwitterAgent.sources.Twitter.channels = MemChannel
	TwitterAgent.sources.Twitter.consumerKey = <KEY>
	TwitterAgent.sources.Twitter.consumerSecret = <KEY>
	TwitterAgent.sources.Twitter.accessToken = <KEY>
	TwitterAgent.sources.Twitter.accessTokenSecret = <KEY>
	TwitterAgent.sources.Twitter.keywords = <KEY WORDS TO CAPTURE FROM TWEET>

	################## SINK #################################
	TwitterAgent.sinks.HDFS.channel = MemChannel
	TwitterAgent.sinks.HDFS.type = hdfs
	TwitterAgent.sinks.HDFS.hdfs.path = hdfs:/user/Flume/tweet
	TwitterAgent.sinks.HDFS.hdfs.fileType = DataStream
	TwitterAgent.sinks.HDFS.hdfs.writeFormat = Text

	TwitterAgent.sinks.HDFS.hdfs.batchSize = 1000
	TwitterAgent.sinks.HDFS.hdfs.rollSize = 67108864
	TwitterAgent.sinks.HDFS.hdfs.rollInterval = 3600
	TwitterAgent.sinks.HDFS.hdfs.rollCount = 0

	#################### CHANNEL #########################
	TwitterAgent.channels.MemChannel.type = memory
	TwitterAgent.channels.MemChannel.capacity = 1000
	#default - TwitterAgent.channels.MemChannel.capacity = 100
	TwitterAgent.channels.MemChannel.transactionCapacity = 1000
	</pre>

	Edit the KEY values as per [your Twitter app] (https://dev.twitter.com/apps)

4. **Start the Flume agent**
	<pre>
	$ flume-ng agent --name TwitterAgent --classpath /Flume/flume-sources-1.0-SNAPSHOT.jar --conf-file /Flume/FlumeTwitter.conf
	</pre>

	Note : `kill -15 <process id of flume agent>` to kill the running flume agent

Setting up Hive
----------------

1. **Build or Download the JSON SerDe**

   A pre-built version of the JSON SerDe is available [here](http://files.cloudera.com/samples/hive-serdes-1.0-SNAPSHOT.jar).

   The `hive-serdes` directory contains a Maven project with a JSON SerDe which enables Hive to query raw JSON data.

   To build the hive-serdes JAR, from the root of the git repository:

   <pre>
   $ cd hive-serdes
   $ mvn package
   $ cd ..
   </pre>

   This will generate a file called `hive-serdes-1.0-SNAPSHOT.jar` in the `target` directory. Upload the file in HDFS location(/user/Flume/)

2. **Upload other files for join operation**

	For Sentiment Analysis, Upload [Time_Zone_Map](http://blog.hubacek.uk/wp-content/uploads/2016/01/time_zone_map.tsv) and [Dictionary](http://blog.hubacek.uk/wp-content/uploads/2016/01/dictionary.tsv) File into the cluster/HDFS

	Save both files in HDFS inorder to create HIVE table.
	<pre>
	$ sudo -u hdfs dfs -put  time_zone_map.tsv  /user/Flume/
	$ sudo -u hdfs dfs -put  dictionary.tsv  /user/Flume/
	</pre>

3. **Create a required HIVE tables and Views**

	Run `hive`, and execute the following commands:

	<pre>
	add jar hdfs:/user/Flume/hive-serdes-1.0-SNAPSHOT.jar;
	CREATE EXTERNAL TABLE tweetraw (
   id BIGINT,
   created_at STRING,
   source STRING,
   favorited BOOLEAN,
   retweet_count INT,
   retweeted_status STRUCT<
      text:STRING,
      user:STRUCT<screen_name:STRING,name:STRING>>,
   entities STRUCT<
      urls:ARRAY<STRUCT<expanded_url:STRING>>,
      user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
      hashtags:ARRAY<STRUCT<text:STRING>>>,
   text STRING,
   user STRUCT<
      screen_name:STRING,
      name:STRING,
      friends_count:INT,
      followers_count:INT,
      statuses_count:INT,
      verified:BOOLEAN,
      utc_offset:INT,
      time_zone:STRING>,
   	in_reply_to_screen_name STRING
	)
	ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe'
	LOCATION '/user/Flume/tweet';
	</pre>

	The table can be modified to include other columns from the Twitter data, but they must have the same name, and structure as the JSON fields referenced in the [Twitter documentation](https://dev.twitter.com/docs/tweet-entities).

	<pre>
	CREATE VIEW tweets_simple AS
	SELECT
	id,
	cast ( from_unixtime( unix_timestamp(concat( '2016 ', substring(created_at,5,15)), 'yyyy MMM dd hh:mm:ss')) as timestamp) ts,
	substring(cast(cast( from_unixtime( unix_timestamp(concat( '2016 ', substring(created_at,5,15)), 'yyyy MMM dd hh:mm:ss')) as timestamp) as string),1,10) created_at_string,
	text,
	user.time_zone,
	source source_raw
	FROM tweetraw;

	CREATE VIEW tweets_clean AS
	SELECT
	id,
	ts,
	created_at_string,
	source_raw,
	text,
	m.country
	FROM tweets_simple t LEFT OUTER JOIN time_zone_map m ON t.time_zone = m.time_zone;


	CREATE VIEW l1 AS SELECT id, words FROM tweetraw lateral view explode(sentences(lower(text))) dummy AS words;

	CREATE VIEW l2 AS SELECT id, word FROM l1 lateral view explode( words ) dummy AS word ;

	CREATE VIEW l3 AS SELECT
	id,
	l2.word,
	CASE d.polarity
	WHEN 'negative' THEN -1
	WHEN 'positive' THEN 1
	ELSE 0 END AS polarity
	FROM l2 LEFT OUTER JOIN dictionary d ON l2.word = d.word;


	CREATE TABLE tweets_sentiment
	STORED AS ORC AS
	SELECT
	id,
	CASE
	WHEN SUM( polarity ) > 0 THEN 'positive'
	WHEN SUM( polarity ) < 0 THEN 'negative'
	ELSE 'neutral' END AS sentiment
	FROM l3 GROUP BY id;


	CREATE TABLE tweet
	STORED AS ORC
	AS
	SELECT
	t.*,
	CASE s.sentiment
	WHEN 'positive' THEN 2
	WHEN 'neutral' THEN 1
	WHEN 'negative' THEN 0
	END AS sentiment
	FROM tweets_clean t LEFT OUTER JOIN tweets_sentiment s ON t.id = s.id;

	CREATE EXTERNAL TABLE tweets_new (id BIGINT,entities STRUCT<hashtags:ARRAY<STRUCT<text:STRING>>>) ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe' LOCATION '/user/Flume/tweetraw';

	CREATE TABLE hashtags_new as select id as id,entities.hashtags.text as words from tweets_new;

	CREATE TABLE hashtag_word_new as select id as id,hashtag from hashtags_new LATERAL VIEW explode(words) w as hashtag;
	</pre>

Visualization
--------------

Use the HIVE tables `hashtag_word_new`  and  `tweets`  for Visualization in Tableau/equivalent software.
