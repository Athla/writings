# How I processed over 1M Entries per request - With Go!
Before we start, a bit of context of who I am and why it matters in this case. I'm a software developer at Notebook Manufacturing Company, working here for the past two years. In the team I'm at right now, I'm the sole developer, responsible for creating, monitoring and maintaining data pipelines, automations, and a monitoring system with Go and Grafana. 
Another point that I may add, in the previous year, I was a intern. And I'm self taught.

## Ok, but why it matters?
Well, I had no senior developer, or any form of guidance from inside that could guide me when I was facing roadblocks or problems I could not figure out on my own, and that's the main reason I'm writting. I think it was a fun thing to do and want to share. It won't be a amazing piece of software, a breakthrough or anything like that, but a reminder that it's possible to build things from ground up. You don't need to be the one in one million 10x developer or something like.

P.S: I use neovim btw.

## One Million Lines
This number may look like a exageration or something like that, and, although I wish it was the case, it isn't. See, when working in the manufactory context, sometimes we don't take into account how many items are needed, and how much data it generates when you gotta keep track of every point of the many manufacturing queues, for each component.
Be it a little screw, be it a last gen CPU, it must be tracked, with timestamps!
So, yes, the amount of data generated is correct, and that's just the start. Here are the other cool points that make the problem cool (in my perspective): 

- The source of this data wasn't compressed or anything like that, so the average request time was close to 2 to 3 minutes on a good day. It could go over 5 minutes if the source was lagging (It's a older than me java system.).
- About the format, I could choose between dealing with a XML response or a CSV response, of course I took the CSV route. I'm no masochist. 
- And there was a need to choose the interval of the data fetching, with a minimum of one hour (No, I don't know why).
- The source had no lazy loading or cache system, so two equal request would generate the same response, with different times.
- And finally, to spicy things up, it's a Microsoft thing, known as SQL Server Reporting Services. Oh, and the database I'm required to use is MSSQL, which is a pain.

That's the whole context, and now the fun part begin.

### The first iteration - DMP

The way I like to approach software is pretty simple. Make it ugly and barely functional -> Keep it ugly and improve functionality -> Still ugly, but better optmized -> Pretty, Optimized and Working.
And then you manager says that the work you did is way too good and the source can't handle it, causing outages. FML.
Anyway, DMP stands for Dumb Mode Protocol, just make it work in the dumbest way you can, which means doing it to barely work.

So, for the first round of it, my goal was simple, authenticate, make the request, parse the data, send to the database. Pretty simple, right? And, in paper, it was, until I discovered that the authentication and authorization method I had to used was ntlmssp, which is a challenge-response auth method that I didn't knew existed. Matter of fact, I had to go into legacy .NET 4 code to find it. I never did C#. 
After going through the legacy code older than me, suffering trying to understand it because, for some reason, who wrote it thought it was a good idea to hide it into 5 layers of abstraction, constructors and OOP things. Not fun, humbling experience. And then I made it work. A hour later I got my user blocked because, apparently, the source has a rate limiting. BUT NO CACHE OR LAZY LOADING.

Well, after all of that, I just need to pass the query paremeters and get the data, no more complications with the source. Ok, let's look up the documentation for the query parameters.
...
At this point, all things considered, you probably guessed it right. Documentation? Is this some kind of exotic food that is only served in the Sillicon Valley?
Anyway, after racking my brain out, I decided to inspect the site/interface of this SQL Server Reporting Services, and, to my joy, and then hatred, it had a way to know the filters and it's values.

Only to know that the ("All") filter in the page, to select, let's say, all the manufacturing lines, is just a abstraction of having all the boxes checked. Ok, let's copy the filters and put it into the querystring, encode it and be happy!

### IT WORKED! Or so I thought.

Remember when I said that my user got blocked? Well, looks like having the admin permissions to perform such tasks, being authorized by C-Suite to do it (In fact, it was requested bt them), wasn't enough to allow me to do multiple requests. My user got blocked, but to my manager it was the same as saying "Nothing happened". Fortunately, I managed to get it unblocked fairly quickly.
In the mean time, I decided to work on the way I would approach the rest of this project. At this point I already had a sample of that, and it lived up to the title of the post. Seeing the seven digits made me question myself if I was able to do it, since I had zero experience with this data volume.
To get my ideas in shape, I hoped into excalidraw and designed what I wanted to do. I knew the concept of worker pools but hadn't implemented it before. So, I read about it, saw few implementations and asked of mine who helped me a lot. He's the senior I didn't had.

After writting it out in pseudo code, I thought to myself:

"Wow, this is pretty neat, it surely won't generate memory leaks with those channels and goroutines, right?"


Ok, it might not have been those exact words, but it was along those lines. After doing the first iteration of it, managing to do the request (without getting blocked), and loading it into memory, I applied the worker pool concept.

And then I faced the BSOD. Funnily enought, it was at the same day of the CrowdStrike strike, I surely didn't think I caused major outages.
Also yes, I gotta use windows to work, but fret not! I use WSL2.

After going through the enourmous stack track of my first stack overflow, I discovered the mistake. When sending the data to the consumer channel, I didn't took into account that some of the data could error, mainly due violation of primary key or simply because, maybe, I wasn't properly handling errors.

Lesson learned, use a error channel to avoid major problems. And handle your errors, be it via a simple string check (ugly, but works), or simply top level log it, and be happy. Since the errors I faced weren't critical, I could keep going.

### The second iteration. Memory leaks.

The amount of memory leaks I generated in this step made me think I was doing C. But it was just major skill issues.
Anyway, you may be thinking to yourself:

"How did you created memory leaks in such a simple process?"

It's simple, I was learning and trying.

The main problem during this step was how I, naively, didn't made sure that the data was being properly insertted, and the amount of violation of primary keys I was getting was violating my memory. This was a problem I knew how to solve, let's cache the data!
Wait how do I create a Unique Identifier for each line, considering each one of them is unique?

Now, that's the part many will laugh at me, because, to be fair, that's the part that cracks me the most. I simply joined the current row of information and parsed it into a hash function, and that hash became my keys in the map.

"Why not Redis?" - You may be asking yourself.

It's simple, bureaucracy. It's not a small company. Matter of fact, many of you are, probably, using a laptop that they manufactured. The simple act of requesting a Redis instance would take, atleast, three work days, four meetings, a sacrifice for the bureaucracy god, and a complete document explaining why and how.
Sooo, yeah, let's go with a simple hash map and pre initialize it before the first run. It will add to the overall loading time but it will be quicker than the request.

By doing that, the overall process improved like it had a new motor, the memleaks stopped, the batches weren't failing everytime, the amount of errors and disconnections decreased as well, pretty nice, right? Right?

By now, you know something got jambled.

### The third iteration. Life could be good.

One point I did not take into account was the fact that, by doing batch inserts, was the validation of data. Here's a simple representation of the flow.

Fetch data -> Check if the the hash of the data exists in the hashmap -> Batch & Insert

And what is the problem with that? Well, what happens if a single insert in my batch fails, would it retry without the entry? If so, how much retries can I have without cluttering the system and losing the advantages of the worker pool implementation?
Only one way to find out! Let's check it.

At this point, I was already running things in a docker container that mimics the production space with limited resources. To add context, this process was running with 1GB of RAM and about 0.5CPU. I could've allocated more resources, but that would simply be brute forcing my way out.
By running this new iteration inside the container, adding some timestamps and logging it out to a file to later analyze. I discovered a increase of about 5 minutes due the amount of retries. This wouldn't work, removing the 'dirty' entry wasn't a option.

### The fourth iteration. Life IS good.

To solve this issue, I increase the amount of workers. I was using about 50 workers or so, and thanks to a randomic guess of a discord user, I increased it to 1000 workers, and made it so that each worker validated each entry in a transaction. Case the transaction failed, I simply rolled it back.
By doing that, I was able to solve the core issue and improve the overall speed of this process in general. Now it was time to put it into prod and monitor it, because, you know, the prod goblins may jamble your software. (They're also known as skill issues, but this name is forbidden.)

Knowing that decreasing the fetch interval of this system, where it was requested to make it near real time (which means fast enough for them to not notice delays), I created a new container, this time with a little more robust to check if it would be able to hold the load, and set the interval to 3 minutes. Considering the mean time to fetch, it would mean that I could have some overlaps, but I really wanted to see what would happen.

I let it run overnight, logging the results to check it out later, and, to my surprise, before the workday ended, I received a call from my manager. Mind you, he's not a tech person or something like that.

"Hey, have 'we' deployed something that interacts with the [System Name I can't Disclose]?"

"Yeah, I was monitoring the data fetching system in real time that was requested. Why?"

"Can you, uhh, stop it? It caused a outage and we can't work here."

This, gadies and lentleman, is whole another level of breaking prod. I, literally, stopped over 20 production lines for a minute or so. Anyway, I stopped the system and everything went back to normal.
The next day, I was asked to increase the interval between fetches to 30 minutes instead of 3, ok, fine I guess. I won't lie, I was a bit sad that I wouldn't be able to see it performing at max speed, but hey, at least I made it work.

With this new system the average update time of the reports that used this data decreased to 8~10 seconds, which caused a lot more of reports of the same source to be requested by managers in the same way. Good work is rewarded with more work!

## Considerations
It was a fun experience, mainly due the fact it made me truly realize how powerful Go really is. Using less memory than google chrome, less storage than a microsoft app and less CPU power than windows calculator, I was able to improve a old process that literally brute forced it's way through (it literally checked each line in the database before inserting. I don't know how the previous person thought it would be a good idea.). It was fun, really.

Anyway, feel free to sahre your thoughts in this whole process, how would you approach and what would you've done differently. Since I have no dev co-workers, I want to know more views on that.

