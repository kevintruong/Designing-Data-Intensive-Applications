# Chapter 1 - Reliable, Scalable and Maintainable Applications

These are my notes from the first chapter of Martin Kleppmann's: Designing Data Intensive Applications.

**Table of Contents**

- [Reliability](#reliability)
    - [Hardware Faults](#hardware-faults)
    - [Software Errors](#software-errors)
    - [Human Errors](#human-errors)
- [Scalability](#scalability)
    - [Describing Load](#describing-load)
- [Maintainability](#maintainability)

* * *

### Introduction 
Three concerns that are important in most software systems are:

- **Reliability:** The system should work correctly (performing the correct function at the desired level of performance) even in the face of adversity.
- **Scalability:** As the system grows(in data , traffic volume, or complexity), there should be reasonable ways of dealing with that growth.
- **Maintainability:** People should be able to work on the system productively in the future.

### Reliability [#](#reliability)

Typical expectations for software to be termed as reliable are that:

- The application performs as expected.
- It can tolerate user mistakes or unexpected usage of the product.
- Performance is good enough for the required use case, under the expected load and data volume.
- The system prevents any unauthorized access and abuse.

Basically, a reliable system is fault-tolerant or resilient. Fault is different from failure. A fault is one component of the system deviating from its spec, whereas a failure is when the system as a whole stops providing the required service to a user.

It's impossible to reduce the probability of faults to zero, thus, it's useful to design fault-tolerance mechanisms that prevent faults from causing failures.

#### Hardware Faults [#](#hardware-faults)

Traditionally, the standard approach for dealing with hardware faults is to add redundancy to the individual hardware components so that if one fails, it can be replaced. *RAID configuration.*

As data volumes and applications' computing demands have increased, there's been a shift towards using software fault-tolerance techniques in preference or in addition to hardware redundancy. An advantage of these software fault-tolerant systems is that: For a single server system, it requires planned downtime if the machine needs to be rebooted (e.g. to apply operating system security patches). However for a system that can tolerate machine failure, it can be patched one node at a time (without downtime of the entire system - a *rolling upgrade)*

#### Software Errors [#](#software-errors)

Software failures are more correlated than hardware failures. Meaning that, a fault in one node is more likely to cause many more system failures than uncorrelated hardware failures.

There is no quick solution to the problem of faults in software, however, different things can help such as:

- Thorough testing
- Process isolation
- Measuring, monitoring, and analyzing system behavior in production.

#### Human Errors [#](#human-errors)

Humans are known to be unreliable. How do we make systems reliable, in spite of unreliable humans? Through a combination of several approaches such as:

- Designing systems in a way that minimize opportunities for error through well-designed abstractions, APIs, and admin interfaces.
- Decoupling the places where people make the most mistakes from the places where they can cause failures. E.g. by providing a fully-featured non-production sandbox environment where people can explore and experiment safely, using real data, without affecting real users.
- Testing thoroughly at all levels: from unit tests to integration tests to manual tests to automated tests.
- Allow quick and easy recovery from human errors, to minimize the impact of failure. E.g. By making it easy to roll back configuration changes, roll out new code gradually ( so bugs do not affect all users).
- Set up detailed and clear monitoring, such as performance metrics and error rates.

We have a responsibility to our users, and hence reliability is very important.

### Scalability [#](#scalability)

Scalability describes the ability to cope with increased load. It is meaningless to say "X is scalable" or "Y doesn't scale". Discussing scalability is about answering the question of "If the system grows in a particular way, what are our options for coping with the growth?" and "How can we add computing resources to handle the additional load?"

#### Describing Load [#](#describing-load)

Load can be described by the *load parameters.* The choice of parameters depends on the system architecture. It may be:

- Requests per second to a web server
- Ratio of reads to writes in a database
- Number of simultaneously active users in a chat room
- Hit rate on a cache.

**Twitter Case-study**

Twitter has two main operations:

- Posting a tweet: 4.6k requests/sec on average, 12k requests/sec at peak.
- Home timeline: A user can view tweets posted by the people they follow (300k requests/sec)

(*Based on data published in 2012:* [https://www.infoq.com/presentations/Twitter-Timeline-Scalability/](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/))

Twitter's scaling challenge is primarily due to *fan-out*. In electronics, fan-out is the number of input gates that are attached to another gate's output. For twitter, each user follows many people, and each user is followed by many people. What is an optimal way of loading all the tweets of people followed by a user? Two operations could happen:

1.  Posting a tweet inserts the new tweet into a global collection of tweets. This collection could be a relational database which could then have billions of rows - *not ideal.*
2.  Maintain a cache for each user's home timeline - like a mailbox of tweets for each recipient user. When a user posts a tweet, look up all the people who follow the user, and insert the new tweet into each of their home timeline caches. Reading the home timeline is cheap.

Twitter used the first approach initially but they now use approach 2. The downside to approach 2 is that posting a tweet requires a lot of extra work.

For twitter, the distribution of followers per user (maybe weighted by how often those users tweet) is a key load parameter for discussing scalability, since it determines the fan-out load.

Note that Twitter is now implementing a hybrid of both approaches. For most users, tweets continue to be fanned out to home timelines at the time when they are posted. However, for a small number of users with millions of followers (celebrities), they are exempted from the fan out.

**Describing Performance**

Once the load on the system has been described, you can investigate what happens when the load increases in the following two ways:

- When you increase a load parameter and keep system resources (CPU, memory, network, bandwidth, etc.) unchanged, how is the performance of the system affected?
- When you increase a load parameter, how much do you need to increase the resources if you want to keep performance unchanged?

Both questions require numbers.

For Batch processing systems like Hadoop, we care about *throughput -* the number of records we can process per second, or total time it takes to run a job on a dataset of a certain size. For online systems, we care about *response time -* the time between a client sending a request and receiving a response.

*Aside* **-** *Latency vs response time:* These two words are often used as synonyms, but they are not the same. Response time is what the client sees: besides the actual time to process a request(*service time),* it includes network delays and queuing delays. Latency is the duration that a request is waiting to be handled - during which it is *latent,* awaiting service.

Response time can vary a lot, therefore it's important to think of response time not as a single value, but as a distribution of values. Even in a scenario where you'd think all requests should take the same time, you get variation: random additional latency could be introduced by a context switch on the background process, loss of a network packet and TCP retransmission, a garbage collection pause etc.

Average Response time of a service is often reported but it is not a very good metric if you want to know your "typical" response time, because it doesn't tell you how many users actually experienced that delay. A better approach is to use *percentiles.*

**Percentiles**

If you take your list of response times and sort from fastest to slowest, then the *median* is the halfway point: e.g. if median response time is 200ms, it means half the requests return in less than 200ms and half take longer than that. Thus, median is a good metric if you want to know how long users typically wait. Median is known as the *50th percentile,* and sometimes abbreviated as *p50.*

Occasionally slow requests are known as outliers. In order to figure out how bad they are, you look at higher percentiles: *95th, 99th and 99.9th* percentiles are common. These are the response time thresholds at which 95%, 99%, or 99.9% of requests are faster than that particular threshold. For example, if the 95th percentile response time is 1.5 seconds, it means 95 out of 100 requests take less than 1.5 seconds, and 5 out of 100 requests take 1.5 seconds or more.

High percentiles of response times (aka. Tail latencies) are important because they directly impact a user's experience.

Percentiles are often used in SLOs (Service Level Objectives) and SLAs (Service Level Agreements). They set the expectations of a user, and a refund may be demanded if the expectation is not met.

A large part of the response time at high percentiles can be accounted for by queuing delays. It basically refers to how a number of slow requests on the server-side can hold up the processing of subsequent requests. This effect is called *Head-of-Line blocking.* Those requests may be fast to process on the server, but the client will see a slow overall response time due to the time waiting for the prior request to complete. *This is why it is important to measure response times on the client side.* Basically, requests could be fast individually but one slow request could slow down all the other requests.

It takes just one slow call to make the entire end-user request slow. *Tail latency amplification.*

If you want to monitor response times for a service on a dashboard, you need to monitor it on an ongoing basis. A good idea is to keep a rolling window of response times of requests in the last 10 minutes. So there could be a graph of the median and various percentiles over that window.

*Note:* Averaging percentiles is useless, the right way of aggregating response time data is by adding histograms.

**Approaches for Coping with Load**

An architecture that is appropriate for one level of load is unlikely to cope with 10 times that load. Different terms that come up for dealing with this include: *vertical scaling*: moving to a more powerful machine, *scaling out/horizontal scaling:* distributing the load across multiple smaller machines. Distributing load across multiple machines is also known as *shared-nothing* nothing. There's less of a dichotomy between both approaches, and more of a pragmatic mixture of both approaches.

There's no generic one-size fits all approach for the architecture of large scale data systems, it's usually highly specific to the application.

An architecture that scales well for a particular application is built around assumptions of which operations will be common and which will be rare - **the load parameters.**

Note that though they are specific to a particular application, scalable architectures are nevertheless built from general-purpose building blocks, arranged in familiar patterns.

### Maintainability [#](#maintainability)

Design software in a way that it will minimize pain during maintenance, thereby avoiding the creation of legacy software by ourselves. Three design principles for software systems are:

*Operability:*

Make it easy for operations teams to keep the system running smoothly.

*Simplicity*

Make it easy for new engineers to understand the system, by removing as much complexity as possible from the system.

*Evolvability*

Make it easy for engineers to make changes to the system in the future, adapting it for unanticipated use cases as requirements change.

**Operability: Making Life Easy For Operations**

Operations teams are vital to keeping a software system running smoothly. A system is said to have good operability if it makes routine tasks easy so that it allows the operations teams to focus their efforts on high-value activities. Data systems can do the following to make routine tasks easy e.g.

- Providing visibility into the runtime behavior and internals of the system, with good monitoring.
- Providing good support for automation and integration with standard tools.
- Providing good documentation and easy-to-understand operational model ("If I do X, Y will happen").
- Self-healing where appropriate, but also giving administrators manual control over the system state when needed.

**Simplicity: Managing Complexity**

Reducing complexity improves software maintainability, which is why simplicity should be a key goal for the systems we build.

This does not necessarily refer to reducing the functionality of a system, it can also mean reducing *accidental* complexity. Complexity is accidental if it is not inherent in the problem that the software solves (as seen by the users) but arises only from the implementation.

*Abstraction* is one of the best tools that we have for dealing with accidental complexity. A good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand façade.

**Evolvability: Making Change Easy**

System requirements change constantly and we must ensure that we're able to deal with those changes.

**Summary**

An application has to meet functional and non-functional requirements.

- ***Functional -*** What is should do, such as allowing data to be stored, retrieved, searched, and processed in various ways.
- ***Nonfunctional -*** General properties like security, reliability, compliance, scalability, compatibility, and maintainability.

**Reliability** means making systems work correctly, even when faults occur.

**Scalability** means having strategies for keeping performance good, even when load increases.

**Maintainability** is in essence about making life better for the engineering and operations teams who need to work with the system.