---
layout: post
title:  "System Design of my social media app"
date:   2023-04-12 22:55:00 +0530
categories: jekyll update
---

Last year me and my friend started a startup. We started with a BNPL app but shortly after that the revised RBI guidelines came to effect and all our plans were thrown into disarray and we had to pivot to a different idea altogether. In a desperate attempt to catch the billion-dollar wave of BeReal, we pivoted to make a BeReal clone for the Indian market (I know this was one of the worst pivots a startup with our circumstances can take). As we started building the app I realized it was not as simple as it seems. Since I was leading the tech, I had to figure out a lot of the stuff. In this article, I will summarize the high-level system design that I followed to build our MVP.

<br>
<div style="text-align: center;">
  <img src="/asset/images/unfiltr_2.webp" alt="App Screenshot" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
</div>
<br>

Let's start with the tech challenges that we needed to solve.
- Feed should load as quickly as possible
- Reactions and comments should also be visible with minimal delay
- Feed should be customized per user (just basic personalization for now)

Remember, this was just an MVP and we only had about a month and a half to roll everything out, so some of the solutions might seem a little hackish but it worked for us. Since this was a read-heavy system we cached most of the stuff. All the posts were updated in follower feeds and directly served from the cache after a couple of joins.

<br>
<div style="text-align: center;">
  <img src="/asset/images/tweet_normal_user.png" alt="Tweet update" style="height: 100%; max-width: 100%; margin: 0 auto;">
</div>
<br>

One drawback with this approach is that if a user has a large number of followers this update can be slow. Therefore to tackle that, what we did was to update the celebrity posts at the time of fetching the user feed. We checked what celebrities the user is following and then added their most recent post in the user feed. In larger systems, there might be a completely separate logic for handling which post should be added (maybe it's the most interesting to the user or with the most number of likes or a mix of the two).

<br>
<div style="text-align: center;">
  <img src="/asset/images/celebrity_user.png" alt="Tweet update" style="height: 100%; max-width: 100%; margin: 0 auto;">
</div>
<br>

For storing reactions and comments we used the same formula. We cached most of the stuff. Whenever an update was made we updated both the SQL and the cache. Normally when the system scales, these SQL updates will need to be batched or else it will overtax the DB. We followed a simple schema for storing these.

<br>
<div style="text-align: center;">
  <img src="/asset/images/reaction_cache_schema.png" alt="Cache Schema" style="height: 100%; max-width: 100%; margin: 0 auto;">
</div>
<br>

Now for customizing the main user feed we kept it simple. Ideally, different ML pipelines work over the interaction data of the user along with other parameters to achieve this. But we just gave a score to different interaction events like tap, share, comment, and like and generated a rank based on the weighted score (just making an MVP here). The interaction data was put in a Kinesis stream and then in intervals of 30 minutes, the data was batched and written into S3 files. Our backend would pick this data at regular intervals and update the post scores.

<br>
<div style="text-align: center;">
  <img src="/asset/images/kinesis_data.png" alt="Kinesis update" style="height: 100%; max-width: 100%; margin: 0 auto;">
</div>
<br>

Apart from this, there were other optimizations like batching the notifications when it exceeds a particular threshold, converting profile images to lower resolution for loading thumbnails, making a Jenkins task trigger at a random time, etc. To keep things streamlined we used Terraform for managing the infra. This was my second time working this it and I was really impressed by how much efficient it was compared to manually tinkering with the GUI.

Though I enjoyed working on the project, we couldn't make it grow. Social media is one of my favorite fields to work in and I hope I get a chance to make something worthwhile in the future.
