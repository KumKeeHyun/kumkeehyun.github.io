---
title: "kafka connect twitter에 Expansions 기능 추가하기"
permalink: posts/bdp-project/twitter-kafka-connect
classes: wide

categories:
  - bdp project
tags:
  - kafka
  - kafka connect
  - twitter api
last_modified_at: 2023-03-18T00:00:00-00:00
---

# TOC
<!--ts-->
- [TOC](#toc)
- [서론](#서론)
- [고쳐야 할 부분 분석](#고쳐야-할-부분-분석)
- [Expansions 기능 추가하기](#expansions-기능-추가하기)
- [결과](#결과)
<!--te-->

# 서론

앞서 프로젝트 구상에서 원본 데이터를 Twitter API를 통해 얻어오기로 했었다. 
사실 이전에 `jcustenborder/kafka-connect-twitter`를 사용했던 경험이 있어서 데이터를 구하는 것에는 큰 걱정을 않았었다.

그런데 데이터를 뽑아보기 위해 테스트로 실행시켰더니 오류가 뿜어져나왔다. 원인은 twitter connector 내부에서 사용하는 Twitter API v1이 지원만료 되었던 것이었다...

멘붕에 빠져 [kafka-connect-twitter](https://github.com/jcustenborder/kafka-connect-twitter) 레포의 issue와 pr를 보다가 동아줄을 발견했다.

<img width="612" alt="image" src="https://user-images.githubusercontent.com/44857109/226104953-9e9610b1-5850-4d37-821c-9dd2f69dd7fb.png">

머지는 되지 않았지만 직접 빌드해서 사용해보니 잘 작동되었다. 

하지만 큰 산이 하나더 기다리고 있었다. 바로 [Expansions](https://developer.twitter.com/en/docs/twitter-api/expansions) 기능을 지원하지 않아서 내가 필요한 유저 세부 정보를 받을 수 없는 것이었다. 어딘가 썩어있는 동아줄을 고쳐야 하는 상황이 되어버렸다.

# 고쳐야 할 부분 분석

내가 필요한 정보는 트윗 내용(text, created_at)과 유저 정보(name, description)이었다. Twitter API v2는 별다른 인자 없이 그냥 트윗 정보를 조회하면 정말 딱 트윗에 대한 정보만 반환해준다. 

이때 author_id는 제공해주지만 해당 id에 대한 세부 정보를 조인하려면 expansions을 사용해야 한다.

```
curl 'https://api.twitter.com/2/tweets/1212092628029698048?expansions=author_id' --header 'Authorization: Bearer $ACCESS_TOKEN'
```

```
{
    "data": {
        "attachments": {
            "media_keys": [
                "16_1211797899316740096"
            ]
        },
        "author_id": "2244994945",
        "id": "1212092628029698048",
        "referenced_tweets": [
            {
                "type": "replied_to",
                "id": "1212092627178287104"
            }
        ],
        "text": "We believe the best future version of our API will come from building it with YOU. Here’s to another great year with everyone who builds on the Twitter platform. We can’t wait to continue working with you in the new year. https://t.co/yvxdK6aOo2"
    },
    "includes": {
        "users": [
            {
                "id": "2244994945",
                "name": "Twitter Dev",
                "username": "TwitterDev"
            }
        ]
    }
}
```

기존 코드에서 검색 조건을 설정하는 부분을 찾아보니, `filterRule`, `tweetFields`에 대해서만 설정하고 expansions와 해당 fields에 대한 설정은 안하고 있었다.

```java
// https://github.com/TouK/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TwitterSourceTask.java#L138
  private InputStream initTweetsStreamProcessing(TwitterApi apiInstance) throws ApiException {
    InputStream twitterStream;
    if (config.filterRule != null) {
      log.info("Setting up filter rule = {}", config.filterRule);
      setFilterRule(apiInstance); // filterRule 설정
      log.info("Starting tweets search stream.");
      TweetsApi.APIsearchStreamRequest builder = apiInstance.tweets().searchStream();
      if (config.tweetFields != null) {
        log.info("Setting up tweet fields = {}", config.tweetFields);
        builder = builder.tweetFields(Arrays.stream(config.tweetFields.split(",")).collect(Collectors.toSet())); // tweetFields 설정
      }
      twitterStream = builder.execute(RETRIES);
    } 
    // ...
    return twitterStream;
  }

// https://github.com/TouK/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TwitterSourceTask.java#L184
  private void processFilteredStreamingTweetResponse(String line) {
    try {
      FilteredStreamingTweetResponse tweetResponse = FilteredStreamingTweetResponse.fromJson(line);
      if (tweetResponse != null) {
        onTweet(tweetResponse.getData()); // tweet 부분만 사용
      }
    } catch (Exception ex) {
      log.error("Exception during TweetsSampleStreamResponse processing - will be skipped", ex);
    }
  }
```

내가 원하는 기능을 추가하려면 우선 expansions와 관련된 objects에 대한 스키마를 추가하고, Filter Stream 조회 조건에 expansions 및 fields를 추가하는 작업이 필요했다.

# Expansions 기능 추가하기

우선 외부에서 동적으로 expansions를 설정할 수 있도록 properties를 추가했다.

```java
// https://github.com/KumKeeHyun/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TwitterSourceConnectorConfig.java#L28
  public static final String TWITTER_BEARER_TOKEN_CONF = "twitter.bearerToken";
  public static final String TWITTER_BEARER_TOKEN_DOC = "Twitter API Bearer token.";
  public static final String FILTER_RULE_CONF = "twitter.filter.rule"; // filterRule
  public static final String FILTER_RULE_DOC = "Twitter rule used in filtering.";
  public static final String EXPANSIONS_CONF = "twitter.expansions"; // expansions
  public static final String EXPANSIONS_DOC = "Expand objects referenced in the payload.";
  public static final String TWEET_FIELDS_CONF = "twitter.fields.tweet"; // tweet fields
  public static final String TWEET_FIELDS_DOC = "Fields that will be returned for tweet.";
  public static final String USER_FIELDS_CONF = "twitter.fields.user"; // user fields
  public static final String USER_FIELDS_DOC = "Fields that will be returned for user.";
  public static final String PLACE_FIELDS_CONF = "twitter.fields.place"; // place fields
  public static final String PLACE_FIELDS_DOC = "Fields that will be returned for place.";
```

다음으로는 expansions에 관련된 objects(user, place)와 최종적으로 사용할 데이터에 대한 스키마를 추가했다.

expansions에는 user, place이외에도 media, tweets, topic 등이 있지만 당장 사용할 일이 없어서 ~~시간절약을 위해~~ 제외했다.

```java
// https://github.com/KumKeeHyun/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TweetConverter.java#L917
  public static final Schema TWEET_USER_SCHEMA = SchemaBuilder.struct()
          .field(User.SERIALIZED_NAME_ID, Schema.STRING_SCHEMA)
          .field(User.SERIALIZED_NAME_NAME, Schema.STRING_SCHEMA)
          .field(User.SERIALIZED_NAME_USERNAME, Schema.STRING_SCHEMA)
          .field(User.SERIALIZED_NAME_DESCRIPTION, Schema.OPTIONAL_STRING_SCHEMA)
          .field(User.SERIALIZED_NAME_LOCATION, Schema.OPTIONAL_STRING_SCHEMA)
          .build();

  public static Struct convert(@Nonnull User user) {
    return new Struct(TWEET_USER_SCHEMA)
            .put(User.SERIALIZED_NAME_ID, user.getId())
            .put(User.SERIALIZED_NAME_NAME, user.getName())
            .put(User.SERIALIZED_NAME_USERNAME, user.getUsername())
            .put(User.SERIALIZED_NAME_DESCRIPTION, user.getDescription())
            .put(User.SERIALIZED_NAME_LOCATION, user.getLocation());
  }

// https://github.com/KumKeeHyun/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TweetConverter.java#L934
  // place...

// https://github.com/KumKeeHyun/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TweetConverter.java#L990
  public static final Schema TWEET_WITH_EXPANSIONS_SCHEMA = SchemaBuilder.struct()
          .field(SERIALIZED_NAME_TWEET, TWEET_SCHEMA)
          .field(SERIALIZED_NAME_EXPANSIONS, TWEET_EXPANSIONS_SCHEMA)
          .build();
  public static Struct convert(@Nonnull Tweet tweet, Expansions expansions) {

    return new Struct(TWEET_WITH_EXPANSIONS_SCHEMA)
            .put(SERIALIZED_NAME_TWEET, TweetConverter.convert(tweet))
            .put(SERIALIZED_NAME_EXPANSIONS,
                    Optional.ofNullable(expansions)
                            .map(TweetConverter::convert)
                            .orElse(null));
  }
```

마지막으로 Filter Stream의 조회 조건에 expansons 및 fields 정보를 추가하도록 수정했다.


```java
// https://github.com/KumKeeHyun/kafka-connect-twitter/blob/master/src/main/java/com/github/jcustenborder/kafka/connect/twitter/TwitterSourceTask.java#L139
  private InputStream initTweetsStreamProcessing(TwitterApi apiInstance) throws ApiException {
    if (config.filterRule != null) {
      log.info("Setting up filter rule = {}", config.filterRule);
      setFilterRule(apiInstance);
    }
    TweetsApi.APIsearchStreamRequest streamRequest = buildStreamRequest(apiInstance);
    log.info("Starting tweets search stream.");
    return streamRequest.execute(RETRIES);
  }

  private TweetsApi.APIsearchStreamRequest buildStreamRequest(TwitterApi apiInstance) {
    TweetsApi.APIsearchStreamRequest builder = apiInstance.tweets().searchStream();
    if (config.tweetFields != null) {
      log.info("Setting up tweet fields = {}", config.tweetFields);
      builder = builder.tweetFields(Arrays.stream(config.tweetFields.split(",")).collect(Collectors.toSet()));
    }
    if (config.expansions != null) {
      log.info("Setting up expansions = {}", config.expansions);
      builder = builder.expansions(Arrays.stream(config.expansions.split(",")).collect(Collectors.toSet()));
    }
    if (config.userFields != null) {
      log.info("Setting up user fields = {}", config.userFields);
      builder = builder.userFields(Arrays.stream(config.userFields.split(",")).collect(Collectors.toSet()));
    }
    if (config.placeFields != null) {
      log.info("Setting up place fields = {}", config.placeFields);
      builder = builder.placeFields(Arrays.stream(config.placeFields.split(",")).collect(Collectors.toSet()));
    }
    return builder;

  }
```

# 결과

다음과 같이 설정하고 실행해 보았다. 

- 조회 조건: chatgpt에 대한 내용이 있고, 영어로 되어있고, 리트윗이 아닌 트윗
- expansions: author_id(유저에 대한 세부 정보), geo.place_id(위치 정보)

```
twitter.filter.rule=(chatgpt) lang:en -is:retweet
twitter.fields.tweet=author_id,created_at,geo,id,text,lang
twitter.expansions=author_id,geo.place_id
twitter.fields.user=description,location
twitter.fields.place=country,country_code,full_name,geo,id,name,place_type
```

```
{"tweet":{"id":"1636711254311178240","created_at":1679057346000,"text":"@GrahamStephan @memdotai  mem it #chatgpt","author_id":"1486708988","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711254311178240"],"reply_settings":null},"expansions":{"users":[{"id":"1486708988","name":"Hyder Zahoor","username":"hyderzahoor1","description":"All suffering in this world is born from an individual's incompetence.","location":"Jammu and Kashmir , India"},{"id":"22716009","name":"Graham Stephan","username":"GrahamStephan","description":"Real Estate Investor, Car Enthusiast, 4M+ Subs on YouTube. \n\nNewsletter - https://t.co/iQGby1s7dc\n\nInsta - https://t.co/LwD4Qgd2eH","location":"Las Vegas, NV"},{"id":"1316495843621511168","name":"Mem","username":"memdotai","description":"Your self-organizing workspace.\n\nGet Mem It for Twitter: https://t.co/PXu0Fqq8k1…\nWebsite: https://t.co/XSDVK00mbv \nCommunity: https://t.co/arZa1ml8IS","location":"Oyster Point, CA"}],"places":null}}
{"tweet":{"id":"1636711255082713088","created_at":1679057346000,"text":"I tried playing a #chess game against #chatgpt. It was hard playing an AI, but not for the reasons one would expect. \n\nI managed to find two strong moves in a row (12. …Rxd2!! and 14. …Nxb1!!), which allowed me to end the game with an unusual checkmate (Qa1#!!). #chesspunks https://t.co/en7hiBFbLY","author_id":"463015398","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711255082713088"],"reply_settings":null},"expansions":{"users":[{"id":"463015398","name":"Dr Patzer \uD83C\uDDFA\uD83C\uDDE6","username":"DrLangstrand","description":"Tweets about chess, politics and occasionally cute animals. #ChessPunks founding member. Team leader of the Chesspunks team: https://t.co/G754uhLn7M","location":"Sweden"}],"places":null}}
{"tweet":{"id":"1636711267271335936","created_at":1679057349000,"text":"“Any sufficiently advanced technology is indistinguishable from magic” Arthur C. Clarke\n\nIt is cute hearing people amazed by linguistic modelling in ChatGPT - it must seem like magic to some people.","author_id":"1413724477","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711267271335936"],"reply_settings":null},"expansions":{"users":[{"id":"1413724477","name":"Andrew.Knitties","username":"AndrewKingDev","description":"Chief Knitter, Tech Art consultant and Tooler - 21 years. Adobe, Unity. Scholar, Reader, Listener, Supporter of workers - always.","location":"At some disputed barricade"}],"places":null}}
{"tweet":{"id":"1636711267728760835","created_at":1679057349000,"text":"@MoAlhaddar @iamNasmo app idea, use Twitter API to fetch account tweets then send them to chatgpt and get a summary.","author_id":"411535875","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711267728760835"],"reply_settings":null},"expansions":{"users":[{"id":"411535875","name":"⌭Jafar Abdulrasoul (جعفر)","username":"JimmarxD","description":"Professional O2 to CO2 convertor","location":"/root"},{"id":"1113539587627003904","name":"Mohammed Alhaddar","username":"MoAlhaddar","description":"Free falling. It's liberating.","location":"Kuwait"},{"id":"746500371841978368","name":"\uD835\uDCDD\uD835\uDCF8\uD835\uDCFD\uD835\uDCF8\uD835\uDCFB\uD835\uDCF2\uD835\uDCF8\uD835\uDCFE\uD835\uDCFC . \uD835\uDCDD\uD835\uDCEA\uD835\uDCFC ✨","username":"iamNasmo","description":"مواطن متواضع يعشق تراب وطنه \uD83C\uDDF0\uD83C\uDDFC \nBSc Infotech (Aus) / MSc Comp Sc.(UK) \nPhD candidate @ AGU\nTraining Authority Delegate PAAET\n\uD83E\uDD3A Ruthless Pirate by Night \uD83C\uDFF4‍☠️","location":"Valhalla, Hall of the Fallen\uD83E\uDD84"}],"places":null}}
{"tweet":{"id":"1636711360598777858","created_at":1679057371000,"text":"@Mappletons @BVLSingler Where do you &amp; @genmon think that ChatGPT’s poetry comes from? I have a hint—it’s not from its soul. It doesn’t speak from its experience, informed by its struggles to make sense of the world. It’s built on the misappropriated works of human poets—words ripped from their hearts.","author_id":"2468094146","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711360598777858"],"reply_settings":null},"expansions":{"users":[{"id":"2468094146","name":"neil turkewitz","username":"neilturkewitz","description":"Arts advocate. Warning about libertarian definitions of freedom that fail to consider asymmetrical power since 1996. Secular humanist writing about religion.","location":null},{"id":"1343443016","name":"Maggie Appleton","username":"Mappletons","description":"Product design @oughtinc. Makes visual essays about UX, programming, and anthropology. Adores digital gardening \uD83C\uDF31, end-user development, and embodied cognition","location":"London, England"},{"id":"181907107","name":"Prof. Dr Beth Singler.","username":"BVLSingler","description":"Anthropologist and Geek thinking about how you think about AI & robots. Assistant Professor in Digital Religions at UZH. She/her. \uD83C\uDF08","location":"University of Zurich"},{"id":"13124","name":"Matt Webb \uD83C\uDF38\uD83C\uDF3C\uD83C\uDF38","username":"genmon","description":"Blogger / FACTORY worker / Open to collabs + product exploration / If this all evaporates then let's meet up over here: https://t.co/y3wPSWUIdJ","location":"London O London"}],"places":null}}
{"tweet":{"id":"1636711368723144704","created_at":1679057373000,"text":"@100xAltcoinGems @GPT4ERC certainly the best \uD83D\uDC8E to do that #gpt4 #gpt #chatgpt #eth #ethereum #openai #levelup #cryptogpt4 #crypto #btc","author_id":"1636324788141535232","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711368723144704"],"reply_settings":null},"expansions":{"users":[{"id":"1636324788141535232","name":"crypto_Emmanuel","username":"Miamalk62139992","description":"","location":null},{"id":"1517796456949817347","name":"100x Altcoin Gems","username":"100xAltcoinGems","description":"100x Altcoin Gem Hunter \uD83D\uDC8E\uD83D\uDE80 DM for Shill & Promo \uD83D\uDCA5\n\uD83D\uDEA8 NO financial advice | DYOR‼\n\n#BTC #ETH #BNB #SOL #AVAX #FTM #CRO #Crypto #NFTs #Metaverse","location":"Metaverse"},{"id":"937901747053514752","name":"GPT-4","username":"GPT4ERC","description":"GPT-4 With each sentence the power is in your hands to Imagine\n\n0x73706a7d4c34b3c70a1cd35030b847a0e11403e0","location":null}],"places":null}}
{"tweet":{"id":"1636711373630750723","created_at":1679057374000,"text":"You Don’t Have to Be a Jerk to Resist the Bots https://t.co/753PeXlUmQ","author_id":"1523055356812926977","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711373630750723"],"reply_settings":null},"expansions":{"users":[{"id":"1523055356812926977","name":"prodigy8","username":"prodigy8_t","description":"IT Managed Services Provider specializing in small business. \nInvesting in IT Managed Services can give your company the flexibility you need to grow.","location":"Philadelphia, PA"}],"places":null}}
{"tweet":{"id":"1636711377988378625","created_at":1679057375000,"text":"@MasterMilkX @MatthewGuz You know I still hear your voice saying “it’s amazing how much you can do with hand-rolled sentence embeddings” regularly. And it truly is. I recently rolled my own","author_id":"17832548","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711377988378625"],"reply_settings":null},"expansions":{"users":[{"id":"17832548","name":"Martin Pichlmair","username":"martinpi","description":"Empowering creatives in the age of AI · CEO @writewithlaika · Co-founder @brokenrules · Associate Professor @ITUkbh · Doctor of Technology (for real!) · he/him","location":"Copenhagen, Earth"},{"id":"429494244","name":"M Charity","username":"MasterMilkX","description":"Just a they/them programmer | PhD Student at NYU Tandon\n\nGithub: https://t.co/EAVesDR1p2\nItch: https://t.co/Sn45HYP9Hf","location":"Brooklyn, NY"},{"id":"250736679","name":"Matthew Guzdial","username":"MatthewGuz","description":"CS Asst. Prof at @ualberta and CIFAR AI chair at @AmiiThinks. AI, Machine Learning, Games, and Computational Creativity | he/him","location":"Edmonton, Alberta"}],"places":null}}
{"tweet":{"id":"1636711383273283591","created_at":1679057377000,"text":"@SamuelIniJosep4 @thecommunitydao Thinking about investing in an exciting #AI project? Give CryptoAI a chance! Undervalued compared to other big names in the business, I see a 50x potential in the short/medium term for $CAI\n\nChart: https://t.co/jFuccAFc1M\n\nTg: https://t.co/wMNtqjEEAJ\n\n#CryptoAI #CAI #ChatGPT #ETH https://t.co/y6w3Jkn9rX","author_id":"1101458425","in_reply_to_user_id":null,"referenced_tweets":null,"attachments":null,"context_annotations":null,"withheld":null,"geo":{"coordinates":null,"place_id":null},"entities":null,"public_metrics":null,"possibly_sensitive":null,"lang":"en","source":null,"non_public_metrics":null,"promoted_metrics":null,"organic_metrics":null,"conversation_id":null,"edit_controls":null,"edit_history_tweet_ids":["1636711383273283591"],"reply_settings":null},"expansions":{"users":[{"id":"1101458425","name":"ladymae barneso","username":"ladymaebarneso","description":"","location":null},{"id":"1509435850144337920","name":"Samuel Ini Joseph","username":"SamuelIniJosep4","description":"Poet, Student, and Markerter.\nA Liverpool fan and a Crypto Enthusiast\nJoin me today @thecommunitydao","location":null},{"id":"1272051409538543617","name":"¢ommunity DAO \uD83C\uDD41\uD83C\uDD45\uD83C\uDD3D","username":"thecommunitydao","description":"REGNAT POPULUS - The People rule.\nUniversal Decentralized C0MM | Governance: https://t.co/G2O4dUjYrY…\n-Join our Discord \uD83D\uDC49 https://t.co/hdOoxspcmY","location":"Metaverse"}],"places":null}}
% Reached end of topic tweets [0] at offset 9
```

드디어 내가 원했던 유저 정보가 트윗 정보에 포함되어 나온다!! 

place.geo 정보를 요청할 수 있어서 나중에 지도에 매핑한 시각화도 해볼 수 있겠다고 좋아했는데, 사용자가 트윗에 따로 지역 정보를 입력하지 않으면 place는 조회되지 않는 것 같다. 유저 정보를 따로 조인해주지 않아도 된다는 사실에 만족하기로 했다.