"My approach is: Detect → Acknowledge → Assess Impact → Prioritize → Communicate → Investigate → Mitigate → Restore Service → RCA → Prevent Recurrence."

Q1. Difference between Incident and Problem Management?

Incident Management focuses on restoring service as quickly as possible when there is an outage or degradation. Problem Management focuses on finding the root cause and preventing the incident from happening again.

Q2. What will you do when PagerDuty alerts at 2 AM?

First, acknowledge the alert and check the impact. Then verify dashboards, logs, Kubernetes pods, and infrastructure health. If required, engage relevant teams, mitigate the issue, restore service, and communicate updates to stakeholders.

Q3. How do you decide P1, P2, P3 severity?

Severity is based on business impact, number of users affected, and service criticality. P1 is a complete outage with major business impact, P2 is a high-impact partial outage, and P3 is a moderate issue with available workarounds.

Q4. What is MTTA and MTTR?

MTTA (Mean Time to Acknowledge) measures how quickly the team responds after an alert is generated. MTTR (Mean Time to Resolve/Restore) measures how long it takes to restore the service after the incident is detected.

Q5. What information must be included in an incident update?

An incident update should include the incident number, impact, affected services, current status, actions taken, teams engaged, and the next update time. Communication should be clear and timely to keep stakeholders informed.

Q6. What is the role of an Incident Manager?

The Incident Manager coordinates the entire incident lifecycle. They manage communications, drive bridge calls, escalate issues, ensure SLA compliance, and coordinate teams until the service is restored.

Q7. How do you handle stakeholder communication during outages?

I provide regular updates with impact, progress, mitigation actions, and expected next steps. I avoid technical jargon and ensure business stakeholders understand the situation and recovery status.

Q8. What would you do if the RCA is not known yet?

I focus on restoring the service first and continue investigating in parallel. I communicate transparently that the root cause is under investigation and provide updates as more information becomes available.

Q9. When do you declare a Major Incident?

A Major Incident is declared when there is significant business impact, critical service disruption, or widespread customer impact requiring urgent attention and cross-functional coordination. Typically P1 and some P2 incidents qualify as Major Incidents.

Q10. Have you handled any production incident? Explain end-to-end.

Yes. I received a PagerDuty alert for high API latency in production. I acknowledged the alert, checked Grafana, logs, and Kubernetes pods, identified a bad deployment, rolled back the release, validated service recovery, communicated updates to stakeholders, closed the incident, and later participated in RCA and preventive action planning.


Incident Management:

Incident Management focuses on restoring the service as quickly as possible and minimizing business impact. The objective is recovery of the service, not necessarily finding the root cause.

Problem Management:

Problem Management focuses on identifying the root cause of incidents and implementing preventive measures. It helps reduce recurring incidents and improves overall service stability.

Change Management:

Change Management ensures that modifications to applications, infrastructure, or configurations are implemented in a controlled and approved manner with minimal risk to production services. This includes approvals, testing, deployment planning, and rollback procedures.
