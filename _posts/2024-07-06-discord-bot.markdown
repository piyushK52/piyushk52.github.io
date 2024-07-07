---
layout: post
title:  "Designing a good enough Discord bot"
date:   2024-07-06 20:44:32 +0530
categories: jekyll update
---

Recently, I worked on a Discord bot that people can use to generate AI images and videos. Here, I will share a quick overview of its system design.


<div style="text-align: center;">
  <img src="/asset/images/discord_bot.png" alt="Discord bot" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
</div>
<br>

The Discord client and most of the FastAPI processes are quite fast, involving only some database operations. The GPU load has been offloaded to third-party APIs, and the results are received through webhooks. However, I noticed some slowdown when multiple users started using the bot simultaneously. Upon further investigation, I realized that only a couple of methods were the bottlenecks in the process, such as image_transform (which downloads ~20 images, resizes them, crops them, and finally uploads them). I shifted all these CPU-intensive methods to Lambda functions. This greatly improved the speed and decreased the load on the VM. Now that both CPU and GPU-intensive processes are highly scalable, the bot can easily handle thousands of users.

The bot structure was also made modular, using cogs for individual commands. Since Discord client threads have a connection timeout, most of the client calls returned immediately from the backend, and the tasks were added in the background. Sample code is shown below.

<div style="text-align: center;">
  <img src="/asset/images/bot_fastapi_code.png" alt="Sample code" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
</div>
<br>


To scale this to millions of users, one would need to shard the client instance across multiple VMs, preferably in an ECS cluster. Similarly, the backend would need to be deployed in an ECS cluster.

As the bot created and stored numerous images and videos, we implemented a simple cron job to schedule S3 bucket cleanup. Whenever a file was created, we injected expiry_time metadata into it, which the cleanup job used to determine whether to delete the file or not.

<div style="text-align: center;">
  <img src="/asset/images/lambda_cleanup.png" alt="Cleanup" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
</div>
<br>