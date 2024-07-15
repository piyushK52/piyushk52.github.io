---
layout: post
title:  "Database locked!"
date:   2024-07-14 20:44:32 +0530
categories: jekyll update
---

I was using sqlite as a local lightweight database for a project and I ran into a weird error "Database is locked" (full trace <a href="/asset/images/database_locked.png" target="_blank">here
</a>). Since I had mostly worked with Postgres and MySQL in my previous projects, this error turned out to be slightly more tricky than what I had thought.

Since I was already using atomic transactions at all the places it is required plus given the fact that sqlite by default works with serializable isolation level, I assumed that the only thing that could be going wrong would be the timeout. That is, some process is holding the database lock for significantly longer and other processes, unable to write to the db, are throwing this error.

<div style="text-align: center;">
  <img src="/asset/images/sqlite_timeout.png" alt="Discord bot" style="max-height: 600px; max-width: 70%; margin: 0 auto;">
</div>
<br>

I updated the timeout for every new connection that was created to 30 seconds. I was positive that no transaction is even remotely reaching this duration, but I was still getting the same "database locked" errors randomly. Goggling stuff I came around this blog <a href="https://fractaledmind.github.io/2023/12/11/sqlite-on-rails-improving-concurrency/" target="_blank">Fractaled Mind</a>. It mentions a very important piece of data that I think is missing from many online forums (and also is not very clear in the documentation). 

<div style="text-align: center;">
  <img src="/asset/images/sqlite_lock.png" alt="Discord bot" style="max-height: 600px; max-width: 90%; margin: 0 auto;">
</div>
<br>

The main issue is that the lock is not acquired when the transaction begins but when the write operation begins. This is very different from how other production databases work, and apart from other shortcomings of SQLite, this makes it a poor choice to be used in production.
<div style="text-align: center;">
  <img src="/asset/images/transaction_lock.png" alt="Transaction Lock" style="max-height: 600px; max-width: 90%; margin: 0 auto;">
</div>
<br>

To fix this issue I made a simple context block that wrapped my atomic transactions, as shown below. The benefit of this approach is that you can independently make your transactions atomic/non-atomic.

<div style="text-align: center;">
  <img src="/asset/images/sqlite_atomic_context.png" alt="Transaction Lock" style="max-height: 600px; max-width: 90%; margin: 0 auto;">
</div>
<br>