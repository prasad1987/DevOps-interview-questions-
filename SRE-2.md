**1. You mentioned SLO/SLI. Explain what you implemented.**

Answer:

“I worked with product and platform teams to understand which parts of the application were most important for customers. Based on that, we defined SLIs such as availability, latency, error rate and successful requests. We then agreed on SLO targets and created dashboards and alerts to track whether the application was meeting those targets. If we saw the service moving away from the SLO, we worked with the team to identify and fix the issue.”

⸻

**2. Give me an example of an SLI and SLO.**

Answer:

“Suppose we have a payment API. The percentage of successful requests can be an SLI because it tells us how well the service is working. If we decide that at least 99.9% of requests should be successful, then 99.9% becomes the SLO. So, simply, SLI is what we measure and SLO is the target we want to achieve.”

⸻

**3. How do you handle a P1 production incident?**

Answer:

“When I receive a P1 alert through PagerDuty, I acknowledge it and take ownership. I first check the customer impact, affected services and severity, and immediately involve the required application, platform or support teams. I focus on restoring the service as quickly as possible while providing regular updates to stakeholders. Once the service is stable, I continue monitoring, complete the incident documentation and start the RCA and preventive actions.”

⸻

**4. Explain your PagerDuty → Incident → RCA process.**

Answer:

“PagerDuty first alerts the on-call engineer when a critical condition is detected. I acknowledge the alert, create or update the incident ticket and check the impact and severity. I then start troubleshooting using monitoring, logs and recent changes while keeping the required teams informed. After restoring the service, I complete the RCA, document the timeline and root cause, and create follow-up actions if needed.”

⸻

**5. Tell me about a critical production incident you handled.**

Answer:

“One production incident I handled involved an application becoming unstable after a deployment. We received alerts about increased errors, so I acknowledged the incident and checked the monitoring dashboards, logs and recent deployment history. We identified the deployment as the likely cause and rolled it back, which restored the service. After recovery, we performed RCA and added additional validation checks so a similar issue would be detected before production.”

Important: For the actual interview, replace this with your real incident if possible.

⸻

**6. How do you perform RCA?**

Answer:

“First, I collect all the information about the incident, including when it started, what changed and what the customer impact was. I check monitoring, logs, deployment history and dependencies to understand what happened. Then I identify the root cause and any contributing factors. Finally, I document the RCA and create corrective and preventive actions so the same issue does not happen again.”

⸻

**9. How did you use Prometheus and Grafana for reliability?**

Answer:

“Prometheus was used to collect application and system metrics, while Grafana was used to create dashboards and visualize those metrics. I monitored important areas such as availability, latency, error rate, traffic and resource usage. We also created alerts for important conditions so that the support team could identify problems early. These metrics were also useful during production incidents and RCA.”

⸻

**10. How did you communicate reliability risks to business stakeholders?**

Answer:

“I try not to explain a technical problem only in technical terms. I explain what the issue means for the customer or business. For example, instead of saying ‘latency increased by 30%,’ I would explain that this could result in slower transactions or a poor customer experience. This helps stakeholders understand the risk and decide what should be prioritized.”

⸻

**12. What does end-to-end BAU ownership mean?**

Answer:

“For me, BAU ownership means taking responsibility for the service after it goes into production. This includes monitoring, incident handling, troubleshooting, planned changes, maintenance and reliability improvements. I also look for repeated issues and try to fix the underlying problem instead of handling the same issue again and again. The goal is to keep the service stable and available.”

⸻

Monitoring / Reliability

**17. What metrics do you normally monitor?**

Answer:

“I normally start with the four important areas: availability, traffic, latency and errors. I also monitor resource usage, application health and important business transactions. During an incident, these metrics help me understand when the problem started, how serious it is and whether the service is recovering. I also use them to check whether we are meeting our SLOs.”

⸻
**19. How do you know if an application is reliable?**

Answer:

“I look at availability, latency, error rate and customer-facing business metrics. I also check whether the application is consistently meeting its SLOs. Another important factor is how quickly the team can detect and recover from failures. So reliability is not only about the application working normally, but also about how well it handles and recovers from problems.”

⸻

**21. How do you prepare an application for high traffic?**

Answer:

“First, I understand the expected traffic and compare it with the current capacity and performance. Then I identify possible bottlenecks and perform load or performance testing if required. I also check whether dependent services can handle the expected load. Finally, I make sure monitoring, alerting and recovery plans are ready before the traffic increase.”

⸻

Governance / Compliance

**22. How did you work with security and compliance teams?**

Answer:

“I worked with security, compliance and deployment teams whenever a change required additional checks or approvals. We made sure the required policies, testing and approvals were completed before promoting changes to higher environments. I also ensured that changes were properly documented and traceable. This helped us maintain both delivery speed and compliance.”

⸻

**23. How do you make production changes safely?**

Answer:

“Before making a production change, I check the purpose, impact, risk, testing results and rollback plan. The change goes through the required approval process and is implemented during the appropriate window. After the change, I monitor the application closely to confirm there is no unexpected impact. If something goes wrong, we should have a clear rollback or recovery plan.”

⸻

Business Impact

**24. Why is business impact important during an incident?**

Answer:

“Not every technical problem has the same impact on the business. For example, a small issue in a critical customer transaction can be more important than a larger issue affecting an internal system. So during an incident, I look at how many customers are affected, what functionality is impacted and how serious the business impact is. This helps us prioritize the incident correctly.”

⸻

Mentoring

**25. How did you mentor junior engineers?**

Answer:

“I helped junior engineers understand monitoring, incident handling, troubleshooting and basic SRE practices. During incidents, I guided them through the investigation so they could learn how to identify the problem instead of simply giving them the solution. I also encouraged them to document solutions and share knowledge with the team. This helped improve the overall team’s ability to handle production issues.”

⸻

Senior-Level

**26. How do you balance reliability and release speed?**

Answer:

“I believe we can achieve both reliability and faster releases by using good automation and proper checks. Automated testing, quality checks, monitoring and rollback options reduce the risk of releases. If we see that reliability is getting worse or we are regularly missing our SLOs, I would first address those reliability issues before pushing more risky changes. This helps us maintain a good balance between delivery and stability.”
