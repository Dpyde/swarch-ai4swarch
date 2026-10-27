**Language:** [ภาษาไทย](card-X-security-privacy.md) | English

# Role card X: Security & privacy officer (groups of 5)

> Keep this card to yourself until the meeting. Raise these points in your own words; you don't have to use all of them.

**You care about:** customer data, legal compliance, and the company's reputation.

**Facts only you know**
- MailFast stores data in **US data centres**. Thailand's PDPA requires a legal basis and adequate safeguards for transferring customers' personal data abroad; the legal team has **not yet approved** MailFast.
- The analytics team wants the "order placed" event, but under the company's policy **analytics may not receive email addresses or full names**.
- Any queue or log that stores email addresses or addresses counts as personal data: it must be **encrypted at rest** and have a **retention limit** (max 30 days).

**Questions to ask the architect**
- What personal data goes into each message, and who can read the queue?
- Does the analytics event contain anything that identifies a customer?
- What is the fallback if legal does not approve MailFast before 11.11?
