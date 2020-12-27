---
title: CS Games 2015 Parallelism Challenge - Introduction
permalink: /csgames-intro/
categories: [CS Games 2015 Parallelism Challenge]
---

![Desktop View]({{ "/assets/img/csgames-logo.png" }})

Last year, I had the the pleasure to be involved with the CS Games. For those who don't know what they are ([and are to lazy to google it](http://lmgtfy.com/?q=cs+games)), the CS Games are a competition that take place every year and pit teams of CS and engineering students against one another in a series of various challenges. These challenges range across hardcore programming projects, gaming and treasure hunting in the host school. There is also some sport contests, believe it or not.

So where did I fit in all of this? Well, for one, I co-organized the parallelism challenge with a [good friend of mine](http://www.jpdeschamps.com). Since this challenge was largely considered to be the most brutal of the competition, I figured it was as good of a topic as any to start my blog.

{%
    include figure.html
    src="/assets/img/achievement.png"
    caption="I shit you not, we actually got this as an award."
%}

So what was this challenge that made one third of the participants rage quit and left the other two third in mental agony?

Well, basically, what the challenge boiled down to was building a highly responsive application that could handle a fairly big amount of requests in a short time interval. To be more specific, the setup was as follows.

We put in place a server that would send problems to be solved to a client. More precisely, every message it sent to a client contained four problems of the same type (more info on the types of problem to be solved [here](https://github.com/faouellet/CSGames2015-Parallelism/tree/master/docs)). When the message was sent, the server started a countdown timer and the client had to respond before the time was up. Failing to do so would result in a point deduction from the client score. It would also most likely cascade into a series of missed deadlines.

Equipped with this problem server, the students had to complete a stub client that we gave them. This client could only read one element from the message the server sent at each time interval, i.e. the problem type ID. It then sent a randomly generated answer back to the server. The competitors then had to complete this client to make it able to handle all the problems the server could throw at it without delays. Sounds easy enough, right?

{%
    include figure.html
    src="/assets/img/ready.jpg"
    caption="Your body might be ready Reggie, but the students' were clearly not."
%}

On a more technical level, the challenge was a C++ only challenge. The framework was coded with the help of the ever useful Boost libraries but we restricted the students to the standard C++ library to implement their solutions. The only place they could (and had to) use Boost was when completing the problems networking part of the client.

In hindsight, having the students complete the networking part of the challenge was probably asking too much of them for an already difficult challenge. Thus, as mea culpa, I will reopen the challenge but this time with the networking part of the challenge already solved. For those willing to try the challenge (and I particularly encourage students planning to take part in the next CS Games to try), you'll find everything you'll need in this [repository](https://github.com/faouellet/CSGames2015-Parallelism). Feel free to submit your solutions by email, we're ready to evaluate you.

Now, try to see if you could come up with a solution within three hours like the previous competitors had to.

**Coming up next month**: I will show you one possible way to not lose your mind to this challenge.
