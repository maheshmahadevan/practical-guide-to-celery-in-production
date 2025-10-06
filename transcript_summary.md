## 🎙️ *A Practical Guide to Celery* — Transcript

**Speaker:** Mahesh Mahadevan, Principal Software Engineer
**Event:** [Python Conference / Internal Tech Talk]
**Duration:** ~33 minutes

---

### [00:00 – 00:03]

**Introduction**

Hi everyone, thanks for the introduction.
I’ll start with a quick background about myself and what we do.

I’m Mahesh, Principal Software Engineer at a SaaS company focused on conversation intelligence. Our product is an AI meeting assistant that helps sales and customer-facing teams accelerate revenue by analyzing conversations.

I’ve been working with Python and Celery for a couple of years now. Today, I’ll walk you through:

1. Basics of Django and Celery
2. How we run Celery in production
3. Our scaling and deployment setup
4. Monitoring and learnings from experience

---

### [00:03 – 00:07]

**Understanding Django and Celery**

Django is a “batteries-included” web framework that helps you go from idea to production quickly.
Celery, on the other hand, is an asynchronous task queue that lets you offload time-intensive or long-running operations from your main request-response cycle.

Celery uses message brokers like RabbitMQ, Redis, or AWS SQS for communication.
In our case, since we are fully on AWS, we use **SQS** as our broker.

Some examples of how we use Celery:

* Video and audio processing
* Generating reports
* Sending emails and notifications
* Running background analytics

It’s a core part of how our backend scales and stays responsive.

---

### [00:07 – 00:10]

**Architecture Overview**

At a high level, here’s what happens:

* The Django application enqueues tasks into SQS.
* Celery worker nodes (running on EC2) pick up tasks from SQS.
* Once tasks are processed, the worker acknowledges completion and the message is deleted from the queue.

We run Celery workers on **AWS Spot Instances** to reduce cost—spot instances are about 70% cheaper than on-demand ones.
All these workers are managed using **Auto Scaling Groups (ASGs)**.

We separate tasks by type using multiple queues:

* **Video processing** → high CPU and memory workers
* **Email/reporting** → smaller, faster workers

Each queue type maps to a specific ASG through AWS tags.
This helps us tune concurrency and resource allocation per task type.

---

### [00:10 – 00:13]

**Scaling Strategies**

We have three types of scaling mechanisms:

1. **Time-based scaling:**
   We scale up during U.S. business hours and scale down during off-hours in India.

2. **Queue depth scaling:**
   Based on how many messages are pending in the queue, we scale up or down workers.

3. **Predictive scaling:**
   AWS provides a predictive scaling feature that uses ML to forecast upcoming load and proactively adds capacity.
   This helps us avoid cold starts, especially during predictable spikes.

The challenge with spot instances is that AWS can terminate them anytime with only a 2-minute warning.
To handle this gracefully, we use a shutdown hook—a cron job that listens for termination notices.
Once triggered, it tells Celery to stop accepting new tasks and finish the ones in progress before shutting down.

For **scale-in** events, we use **EventBridge → Lambda → SSM** to gracefully terminate instances by sending SSM commands to stop workers cleanly.

---

### [00:13 – 00:16]

**Production Metrics**

Here’s what our setup looks like at scale:

* 150+ unique Celery tasks
* 50 queues mapped to 50 Auto Scaling groups
* Up to **20,000 messages** in the queue during peak hours
* Between **100–300 workers** scaling dynamically
* Around **10K to 1M tasks processed per day**

This elasticity helps us handle fluctuating workloads efficiently without overspending.

---

### [00:16 – 00:19]

**Deployment System**

For deployments, we initially had a simple Fabric script that SSHed into each instance and ran updates sequentially.
This took nearly two hours to roll out changes across all queues.

We later combined **Fabric** with **AWS CodeDeploy**.
Now Fabric builds versioned artifacts, uploads them to S3, and triggers CodeDeploy, which updates all ASGs in parallel.
Deployment time dropped from two hours to about **5–7 minutes**.

We also track deployed versions using **Grafana dashboards** that query instance metadata through **Athena** to ensure all nodes are running the correct build.

---

### [00:19 – 00:23]

**Monitoring Setup**

We tried Celery Flower and the Prometheus exporter, but both had limitations with SQS as a broker.
So, we built custom monitoring using **Grafana + CloudWatch + Athena**.

Our dashboards include:

* Queue depth (message count per queue)
* Age of oldest message (to detect lag)
* Auto Scaling events
* Task latency (P50, P99, and max durations)

We set alerts for:

* Oldest message exceeding a threshold (e.g., 180 seconds for meeting join tasks)
* Queue depth beyond a defined limit

These alerts have helped us catch slow queues or scaling mismatches before they impact users.

---

### [00:23 – 00:26]

**War Stories and Learnings**

Some key learnings from production:

1. **Scaling efficiency:**
   Predictive scaling saved us from cold starts and significantly improved responsiveness.

2. **Deployment speed:**
   Moving to parallel CodeDeploy reduced our rollout time drastically.

3. **Spot instance handling:**
   EventBridge + Lambda + SSM made shutdowns smooth and reduced task interruptions.

---

### [00:26 – 00:28]

**Handling Long-Running Tasks**

Two key Celery settings that matter a lot:

* **Concurrency:** how many tasks a worker runs simultaneously.
* **Prefetch multiplier:** how many tasks Celery fetches in advance per worker.

For long-running tasks, it’s important to keep the prefetch multiplier low.
We set:

```python
task_acks_late = True
worker_prefetch_multiplier = 1
```

This ensures that workers only fetch what they can actually process.

We also tuned the **SQS visibility timeout**—that’s the time a message stays hidden after being picked up.
If a worker takes longer than this to finish, the message reappears, leading to duplicate execution.
We set it to the P99 task time plus a few minutes of buffer.

---

### [00:28 – 00:31]

**Ongoing Challenges**

Even with this setup, we still face some challenges:

* Handling incomplete tasks during unexpected spot terminations.
* Detecting and retrying unacknowledged messages efficiently.
* Profiling memory and performance of Celery tasks at scale.
* Managing graceful scaling with containerized workers.
* Lack of good native monitoring tools for Celery + SQS setups.

We’re continuously iterating to improve stability and observability.

---

### [00:31 – 00:33]

**Q&A Session**

**Audience:**
You mentioned using `task_acks_late`. We faced memory issues when enabling it in Kubernetes. Did you see the same?

**Mahesh:**
Yes, that’s quite common. Celery isn’t great at handling memory-intensive workloads.
We used tools like **memory-profiler** and **cProfile** to find leaks and added explicit garbage collection in some cases.

**Audience:**
We’ve seen connection drops in RabbitMQ during scale-ups. Any tips?

**Mahesh:**
We use SQS instead of RabbitMQ, so our setup’s a bit different, but happy to discuss offline if you want to troubleshoot.

---


