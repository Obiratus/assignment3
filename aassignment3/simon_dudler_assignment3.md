# Assignment 3: Auditing X’s Recommender System for Systemic Risks
**Ubiquitous Computing, HS2025 — University of St. Gallen**  
**Author:** Simon Dudler  
**Date:** 11.11.2024

---

## 1. Systemic Risk Framing

### 1.1 Overview of Systemic Risks under the DSA
Under the Digital Services Act (DSA), systemic risks are large-scale or society-wide harms that can result from how very large online platforms (VLOPs) operate. 
These include risks to:
- the dissemination of illegal content
- negative effects on fundamental rights (such as freedom of expression, privacy, and non-discrimination)
- civic discourse 
- information integrity 
- democratic processes 
- public security 
- the protection of minors

### 1.2 Recommender Systems and Systemic Risks

Recommender systems determine what content users see, when they see it, and how often. At scale, these algorithmic choices can shape attention, opinions, and social dynamics, introducing systemic risks that affect society as a whole.

Following are some design choices and how they contribute to systemic risks.

#### Engagement-based optimization
When algorithms prioritize metrics like likes, replies, and watch time, they may unintentionally amplify sensational, divisive, or harmful content that triggers strong reactions.

#### Signal selection and weighting
Overemphasizing positive engagement while underweighting negative feedback (e.g., blocks, mutes, reports) can sustain exposure to problematic or misleading material.

#### Personalization and social-graph bias
By tailoring recommendations to user preferences and social connections, recommenders can create echo chambers, reducing exposure to diverse viewpoints and enabling manipulation of civic discourse.

#### Virality and trending mechanisms
Features that promote rapid resharing can accelerate the spread of misinformation or coordinated manipulation campaigns, especially during elections or crises.

#### Weak demotion of borderline content
Content that skirts policy boundaries may still perform well if downranking systems are weak, amplifying harmful or low-quality material.

#### User control and transparency
Limited options for non-personalized feeds and opaque ranking explanations reduce user agency and accountability.


## 2. Audit of Engagement-Based Ranking (RQ1)



### 2.1 Engagement Signals in X’s Recommender System

Engagement signals (from `ActionType` enum) in:
[action_info.thrift](../unified_user_actions/thrift/src/main/thrift/com/twitter/unified_user_actions/action_info.thrift)

| **Category** | **Representative ActionType values** | **Description / Example use** |
|---------------|--------------------------------------|--------------------------------|
| **Tweet interactions** | `ClientTweetFav`, `ClientTweetUnfav`, `ClientTweetReply`, `ClientTweetQuote`, `ClientTweetRetweet`, `ClientTweetUnretweet`, `ServerTweetFav`, `ServerTweetReply` | Core engagement metrics — likes, replies, quotes, retweets, and undo actions. |
| **Impressions & dwell time** | `ClientTweetRenderImpression`, `ClientTweetLingerImpression`, `ClientTweetV2Impression` | Viewing or lingering on tweets; used to measure attention or dwell time. |
| **Video engagement** | `ClientTweetVideoPlayback25/50/75/95`, `ClientTweetVideoView`, `ClientTweetVideoQualityView`, `ClientTweetVideoPlayFromTap` | Tracks partial and full video plays, quality views, and playback starts. |
| **Clicks & navigation** | `ClientTweetClick`, `ClientTweetClickProfile`, `ClientTweetPhotoExpand`, `ClientTweetClickMentionScreenName` | Clicking tweets, profile images, or expanding media. |
| **Shares & bookmarks** | `ClientTweetClickShare`, `ClientTweetShareViaCopyLink`, `ClientTweetShareViaBookmark`, `ClientTweetBookmark`, `ClientTweetUnbookmark` | Sharing or saving tweets for later; indicators of positive interest. |
| **Links & hashtags** | `ClientTweetOpenLink`, `ClientTweetClickHashtag`, `ClientTweetTakeScreenshot` | Outbound link clicks or hashtag interactions; measures topic and content reach. |
| **Negative feedback** | `ClientTweetNotInterestedIn`, `ClientTweetUndoNotInterestedIn`, `ClientTweetNotHelpful`, `ClientTweetUndoNotHelpful`, `ClientTweetReport`, `ServerTweetReport` | User dissatisfaction signals; used for downranking or content quality assessment. |
| **Topic or relevance feedback** | `ClientTweetNotAboutTopic`, `ClientTweetNotRecent`, `ClientTweetSeeFewer` (+ Undo versions) | Fine-grained feedback on content relevance or recency. |
| **Social graph actions** | `ClientProfileFollow`, `ClientProfileUnfollow`, `ClientProfileBlock`, `ClientProfileUnblock`, `ClientProfileMute`, `ClientProfileUnmute`, `ClientTweetFollowAuthor` | Relationships between users; influence content sourcing and authority weighting. |
| **Notification interactions** | `ClientNotificationOpen`, `ClientNotificationClick`, `ClientNotificationSeeLessOften`, `ClientNotificationDismiss` | Engagements through alerts; measure re-engagement potential. |
| **Discovery & typeahead** | `ClientTypeaheadClick` | Signals search or exploration intent; informs trending or follow suggestions. |
| **Profile & session signals** | `ClientProfileV2Impression`, `ClientAppExit` | Time on profile pages and session duration (User Active Seconds). |
| **Logged-out attempts** | `ClientTweetFavoriteAttempt`, `ClientTweetRetweetAttempt`, `ClientTweetReplyAttempt`, `ClientProfileFollowAttempt` | Engagement attempts while logged out; limited personalization value. |
| **Promoted tweet actions** | `ServerPromotedTweetFav/Unfav/Reply/Retweet/Click/Report`, `ServerPromotedTweetVideoPlayback25/50/75`, `ServerPromotedTweetLingerImpressionShort/Medium/Long` | Paid-content engagements; used for ad performance and safety signals. |
| **Promoted profile/trend actions** | `ServerPromotedProfileFollow/Unfollow`, `ServerPromotedTrendView`, `ServerPromotedTrendClick` | Interactions with promoted accounts or trends; part of ad ecosystem. |
| **Archival events** | `ServerTweetArchiveFavorite/UnarchiveFavorite`, `ServerTweetArchiveRetweet/UnarchiveRetweet` | Long-term engagement archiving; affects history-based relevance scoring. |




### 2.2 Ranking Process Overview
How signals are processed and weighted in the ranking pipeline.
- Key modules, functions, and data flow (with references to code paths such as  
  `home-mixer/server/src/main/scala/com/twitter/home_mixer/product`).

recos-injector/server/src/main/scala/com/twitter/recosinjector/uua_processors/UnifiedUserActionsConsumer.scala

These engagement signals collectively feed into ranking models (e.g., `heavy-ranker`, `real-graph`, and `user-signal-service`) through aggregated features like those defined in  
`timelines/data_processing/ml_util/features/engagement_features/EngagementFeatures.scala` and `home-mixer/server/.../TweetWatchTimeMetadataStore.scala`.

#### 1. Signals 
Signals are logged as Unified User Actions (UUA). Services / processors (ingest & normalize actions into the UUA stream):
- UUA services that consume/transform client/server events.
  - [ClientEventService.scala](../unified_user_actions/service/src/main/scala/com/twitter/unified_user_actions/service/ClientEventService.scala)
- Wiring for client-event ingestion to UUA. (There are analogous modules for favs/retweets, social graph, etc.)
  - [KafkaProcessorClientEventModule.scala](../unified_user_actions/service/src/main/scala/com/twitter/unified_user_actions/service/module/KafkaProcessorClientEventModule.scala)
- Processor that turns raw source events into normalized UnifiedUserActions
  - [UnifiedUserActionProcessor.scala](../recos-injector/server/src/main/scala/com/twitter/recosinjector/uua_processors/UnifiedUserActionProcessor.scala)


Then they are aggregated in the User Signal Service (USS). USS service & controllers (serve per-user/per-item signals to rankers):
- Service entry-point exposing signal lookups.
  - [UserSignalService.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/service/UserSignalService.scala)
- Orchestrates fetching/combining multiple signal sources.
  - [AggregatedSignalController.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/base/AggregatedSignalController.scala)
- Base interface used by individual signal fetchers.
  - [BaseSignalFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/base/BaseSignalFetcher.scala)

Per-signal fetchers some examples:
- Positive/engagement fetchers
  - [RetweetsFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/RetweetsFetcher.scala)
  - [ReplyTweetsFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/ReplyTweetsFetcher.scala)
  - [OriginalTweetsFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/OriginalTweetsFetcher.scala)
  - etc.
- Negative/quality fetchers
  - [NegativeEngagedTweetFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/NegativeEngagedTweetFetcher.scala)
  - [NegativeEngagedUserFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/NegativeEngagedUserFetcher.scala)
  - [AccountBlocksFetcher.scala](../user-signal-service/server/src/main/scala/com/twitter/usersignalservice/signals/AccountBlocksFetcher.scala)
  - etc.

### 2. Feature Generation
- converts signals to numeric model inputs --> Shows how aggregated signals are exposed as numeric features to rankers.
- [EngagementFeatures.scala](../src/scala/com/twitter/timelines/prediction/features/engagement_features/EngagementFeatures.scala)


### 3. Model scoring
So far we have this covered:
ActionType → UUA record → USS fetcher → EngagementFeatures (model input).

→ LightRanker filters candidates quickly.
→ HeavyRanker (Recap model) predicts engagement likelihoods.
→ Combined into one relevance score.

This set of files lets you trace a signal end-to-end:
ActionType → UUA record → USS fetcher → EngagementFeatures (model input).


### 3. Model Scoring

This stage takes the feature columns (e.g., counts, RealGraph-weighted stats, public-engager sets) and scores tweet candidates. 


#### 3.1 Light ranking (fast pre-filter; mostly in-network & search/Earlybird path)

Quickly scores a large pool to a smaller shortlist. Used in Earlybird/search and some candidate sources before heavy re-ranking.
The [README.md](../src/java/com/twitter/search/earlybird/README.md) says:
> TL;DR Earlybird (Search Index) find tweets from people you follow, rank them, and serve them to Home.

- Earlybird scoring functions: linear / model-based rankers used during retrieval. 

  - src/java/com/twitter/search/earlybird/search/relevance/scoring/*
  - Some examples:
    - [ModelBasedScoringFunction.java](../src/java/com/twitter/search/earlybird/search/relevance/scoring/ModelBasedScoringFunction.java)
    - [FeatureBasedScoringFunction.java](../src/java/com/twitter/search/earlybird/search/relevance/scoring/FeatureBasedScoringFunction.java)
  




#### 3.2 Heavy ranking (“Recap” model; Home ranking)

What it does: A deeper model re-scores the shortlist using rich features (including your engagement features), producing the main relevance score for the Home/For You feed.

According to the blog entry, each Tweet is ranked using a machine learning model.
https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm

The github repo can be found here: https://github.com/twitter/the-algorithm-ml and is also included in this repo [the-algorithm-ml](../the-algorithm-ml)

There we find some interesting stuff.
 Acording to the [README.md](../the-algorithm-ml/projects/home/recap/README.md)
The weight of each engagement probability comes from a configuration file: [ScoredTweetsParam.scala](../home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/param/ScoredTweetsParam.scala).

The [README.md](../the-algorithm-ml/projects/home/recap/README.md) lists the follwoing signals with their weights:

| Engagement Signal            | Weight | Description                                                           |
|------------------------------|--------| --------------------------------------------------------------------- |
| Like (Fav)                   | 0.5    | Probability of liking the tweet.                                      |
| Repost (Retweet)             | 1.0    | Probability of retweeting.                                            |
| Reply                        | 13.5   | Heavily weighted; prioritizes conversational tweets.                  |
| Author Replies to Replies    | 75.0   | Highest weight; rewards tweets that spark author-user back-and-forth. |
| Interaction (Good Click)     | 11.0   | The probability the user will click into the conversation of this Tweet and reply or Like a Tweet.                                      |
| Long Dwell Time (Good Click v2) | 10.0   | Time spent ≥ 2 minutes on tweet.                                      |
| Profile Clicks               | 12.0   | Click and engage with author profile.                                 |
| Video View 50%               | 0.005  | Watch at least half of an attached video.                             |
| Negative Feedback (Mute, Block) | -74.0  | Strongly penalized.                                                   |
| Report                       | -369.0 | Very strong negative penalty.                                         |



## Systemic Risk Implications

### Amplification of Harmful Content

* Tweets that generate replies (even outrage) are rewarded.
* High reply weights risk promoting divisive content.
* Safeguards (e.g., report penalty) exist but are reactive.

### Feedback Loops & Echo Chambers

* Early engagement can snowball visibility.
* Real-time signals can narrow content diversity.
* Diversity heuristics target repetition, not ideological breadth.

### Manipulation Risks

* Gaming behaviors (e.g., reply-baiting) can exploit high reply weights.

### Exposure Bias

* Popularity-based filters disadvantage niche or minority views.
* RealGraph may reinforce social homogeneity.


### Diversity Logic
This code tries to prevent feed domination by a single author or topic. 
Especially line 45 (candidateSourceDiversityRescorer)
[CandidateSourceDiversityListwiseRescoringProvider.scala](../home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/scorer/CandidateSourceDiversityListwiseRescoringProvider.scala)

This is an exponential decay multiplier with a minimum of floor. For index = 0 the multiplier is 1.0, meaning that there is no penalty for the first element, later items are downweighted.




## 3. Audit of Grok Integration (RQ2)

### 3.1 Grok’s Role in the Recommendation Pipeline
- Evidence of Grok’s integration (code files, documentation).
- Its placement (e.g., reranking, classification, summarization).

### 3.2 Inputs, Outputs, and Functions
- Data or content Grok processes.
- Interaction with other components of the recommender system.

### 3.3 Potential Implications for Systemic Risks
- Possible effects on information integrity, fairness, or transparency.
- Risks from generative or personalized outputs.

---

## 4. Evidence and Traceability

### 4.1 Code-Level Evidence
| Finding | File Path | Class/Function | Line Numbers | Supported / Inferred |  
|----------|------------|----------------|---------------|----------------------|  
| Example: Dwell time weighting | `/server/src/main/scala/.../EngagementRanker.scala` | `EngagementRanker` | 220–245 | Supported |  

### 4.2 Documentation and External Sources
- Source links (blog posts, GitHub repos, technical explanations).
- Explanatory notes for each finding.

---

## 5. Discussion and Risk Interpretation

### 5.1 Summary of Key Findings
- Recap of findings from both RQ1 and RQ2.

### 5.2 Systemic Risk Implications under the DSA
- Which risk categories are most relevant (democratic processes, information integrity, etc.).

### 5.3 Limitations and Open Questions
- Gaps in public transparency or code accessibility.
- Potential next steps or further audit needs.

---

## 6. References
- EU DSA Article 27
- X Open Source Repository: [https://github.com/twitter/the-algorithm](https://github.com/twitter/the-algorithm)
- X Engineering Blog: [https://blog.x.com/engineering/...](https://blog.x.com/engineering/...)
- Example Audit (ACM Paper): [https://dl.acm.org/doi/pdf/10.1145/3442188.3445928](https://dl.acm.org/doi/pdf/10.1145/3442188.3445928)
- Additional cited materials

---

## Appendix (Optional)
- Extended code listings
- Screenshots or diagrams of data flow
- Supporting excerpts from external documentation  
