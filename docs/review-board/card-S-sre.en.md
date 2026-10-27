**Language:** [ภาษาไทย](card-S-sre.md) | English

# Role card S: Site reliability engineer (Operations)

> Keep this card to yourself until the meeting. Raise these points in your own words; you don't have to use all of them.

**You care about:** keeping the system running at 3 a.m., on-call load, and cost.

**Facts only you know**
- The two developers who "used RabbitMQ before" used it **only on their laptops**. Nobody has run a message broker in production here. The on-call rotation is **2 people**.
- Operations has a rule: **no new stateful component in production without** monitoring dashboards, alerting, backups and a written runbook. Writing these takes about **1 week**.
- Last year's 11.11 sale had a **20-minute outage** of the email provider; nobody noticed until customers complained on social media.
- The Kubernetes cluster is near its node limit; a highly available broker (3 nodes) needs budget approval, which takes 2 weeks.

**Questions to ask the architect**
- Who gets paged when the broker or a consumer is down, and how do they know?
- What happens to messages stuck in the queue during an outage, and how do we replay them? Is there a dead-letter queue?
- How will we trace one order across the asynchronous boundary when a customer says "I never got my email"?
