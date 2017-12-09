# Blog Entry

*by Group E (Alexander, Danny, Kim)*

## Abstract

It is widely known that MongoDB can suffer from performance issues when it has to process too much data. This often leads to high CPU usage, memory spikes and overall degraded system performance, which leads to a poorer user experience. By analyzing the MongoDB log, modifying Mongo’s indexing and isolating and repairing the whole service, the major MongoDB performance problems can be eliminated. This results in a far more stable system environment, and will ultimately result in better response times system-wide.

## The problem

Throughout a project that we were building, we decided to use MongoDB as database, since it was json-based, known to be fast, and ease of use. We had used the database to smaller projects and hadn’t experienced any issues at that time. This project was bigger. It was supposed to consume 30.000.000 items through 3 months. That was another reason why we chose MongoDB; it was document-based, it would (supposedly) not consume a lot of disk space, and it should be easy to append a new document to the database without having duplication errors. 
It was straightforward to insert items, but the CPU utilization that was used to consume all these items, which were posted by a simulator, was crazy. Nonetheless unexpected. [This picture](https://github.com/KIMB0/ufo-blog-entry/blob/master/serverHog.png) is just a short peak at the server process tree using HTOP (a utility to show running processes in Linux).
In the beginning we didn’t knew this. At first glance when the simulator was started there were no issue on the server. It was running fine and it consumed the items rightfully. Then after consuming approximately 600-700.000 items, the server suddenly stopped responding to requests. SSH to the server was not possible and we had to do a hard restart from the Hetzner.com interface, since it is a Virtual Machine. When the server was up and running again, there were missing all previously consumed items in the database. 
We still don’t know why this happened and we were not qualified with MongoDB enough to figure it out. You can read our [Post Mortem Analysis](https://github.com/KIMB0/LSD_frontend/blob/master/Documents/Post_Mrtem_Analysis_GroupE.pdf), where we describe our server going down and what we suspected it to be. 
Performance troubleshooting is one of the biggest challenges developers face today. One of the major complaints about performance troubleshooting is utilization of resource. When the CPU is running at 100% all the time, even after the hard restart, we had to figure out what was causing the trouble. Was it the MongoDB or other processes? Since the above link gives a clear picture of the issue with MongoDB and the utilization, we suspected that to be the issue for the server being slow at consuming messages, since all processes had to fight over the CPU, but MongoDB was eating all of it by its own.

## Why was it a problem?

The performance issue from MongoDB is a big problem in itself. The database is usually a vital part of any applications of a reasonable size. In our Hackernews project, this was no different. As mentioned earlier, a gigantic amount of items were to be consumed and stored in our database. These items were split into two categories - “items” and “users”. This meant that the data was essentially what powered our system. Inability to store this data would result in the application not having anything valid to show.
So in itself, any problems concerning the database would be bad. We just happened to have even more problems coming our way because of this. Why? Because the MongoDB performance issues didn’t just affect its own processes and the data within. It proceeded to affect about every aspect of our server. The server was packed with important programs and tools to run, log, store, and monitor our entire application. We mentioned earlier that MongoDB “stole” a lot of CPU power trying to consume the big payload of data thrown at us, and since our MongoDB instance resided on the same server as everything else, every other program and tool would have little to no processing power to do its own job. This meant that the server became unstable, it crashed frequently, it became excruciatingly slow as it had almost no power left to run our application. 
There was also another major problem with the way MongoDB behaved. Critical, in fact. It was the fact that we mentioned early: A large portion of our consumed database entries completely vanished. No sign or trace of the items were left, and we were essentially back to square one in terms of amount of data stored. It’s not exactly a secret that losing data is bad or even critical. Neither is it a secret that losing 600.000-700.000 items is extremely critical. If our application was distributed and contained unsimulated users and data, losing it all without being able to retrieve it could have had big consequences for us, the users, and the potential application customer(s).

## Modifying the settings

After discovering the massive memory hog, we chose to isolate our MongoDB instance. That meant backing up the current data and applying that data to a cloud machine running MongoDB as its only purpose. After we did this, our primary server suddenly started to [look healthy again](https://github.com/KIMB0/ufo-blog-entry/blob/master/healthyServer.png).
Afterwards it was time to examine the MongoDB logs to find bottlenecks and then eliminating these. We noticed in the logfile that some of the queries made by MongoDB took up to two seconds to process. That might not seem like a lot in itself, but with such a heavy payload being thrown at us, two seconds is precious time to lose. We suspected that the ineffective queries were due to MongoDB indexing every entry with its own unique “_id” field. This can affect performance, mainly because this field includes timestamp and different values that took awhile for the Mongo instance to parse. We ran a command that made MongoDB index every entry with our own id field attached to items in the database. This was just a simple integer value, so we expected that indexing this value would be much more effective.
Profiling was also activated, meaning that MongoDB would begin to store information about exceptionally long/ineffective queries. This would make it even easier to detect bottlenecks.
Finally we ran MongoDBs repair tool, and completely restarted the whole service in hope to free up some memory.

## Result

It lead to a great performance boost now that the server had enough memory to run other processes than MongoDB. Although this just meant that the new isolated MongoDB server had a process tree similar to the application server before the encapsulation happened. The important difference was that the new server was dedicated to this, and it wasn’t the end of the world that it used all of its memory to serve MongoDB, as long as the primary server now had room to breathe.

The new indexing is where we got the real results. After reindexing and restarting the MongoDB service, the average CPU usage from MongoDB went from ~99% to about ~30% instead. A screenshot can be found [here](https://github.com/KIMB0/ufo-blog-entry/blob/master/healthyMongo.png). That’s a huge decrease in processing power. The improvement could instantly be seen on our Rest API too, as the loading time on standard queries were more than halved.

In conclusion, commanding MongoDB to index one of your own values can potentially improve the performance by a landslide. Restarting and repairing the service once in a while in order to let the service breathe, shouldn’t go unnoted either. These points are what worked the best for us. There are tons of settings and configurations to tweak in MongoDB, and what worked for us might not work for you. As mentioned, performance troubleshooting is one of the most difficult tasks for operators/maintainers.

Our advice remains: try altering some of the common settings as mentioned here, check the logs and system configurations to find potential bottlenecks to improve.
We recommend users interested in MongoDB to start off slow. Take the time to read important parts of the documentations, and also hear what other users have to say. This will definitely pay off in the long run. Running MongoDB out of the box is easy and quick, and might work just fine if your suspected dataset is of relatively small size, but if you have intentions to scale up, your whole application can take the hit if your MongoDB is incorrectly set up.

## References

- https://news.ycombinator.com/item?id=3202959
- https://plg.uwaterloo.ca/~migod/research/beckOOPSLA.html
- https://lemire.me/blog/rules-to-write-a-good-research-paper/
- https://docs.mongodb.com/manual/administration/analyzing-mongodb-performance/
- https://www.sitepoint.com/7-simple-speed-solutions-mongodb/
