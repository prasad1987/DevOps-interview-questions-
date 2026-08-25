**1. Production application is down. What will you do?**
   
First, I will acknowledge the incident and check monitoring alerts and dashboards.
I will identify the impact using availability, error rate, latency, and transaction failure metrics.
I will check application logs, Kubernetes pods, infrastructure, database, and dependencies.
If required, I will rollback recent changes or apply a temporary fix.
After recovery, I will perform RCA and implement preventive actions.

**2. How would you troubleshoot high latency in a production application?l**
   
I will start by checking latency, throughput, error rate, and infrastructure metrics.
I will identify whether the issue is from application, database, network, or external dependency.
I will analyze logs, APM traces, slow queries, CPU, memory, and network utilization.
I will optimize the root cause and validate performance after changes.
I will document RCA and add monitoring improvements.

**3. Tell me about a major production incident you handled.**
   
I handled a production issue where application performance degraded due to a backend dependency problem.
I checked dashboards, logs, and application metrics to identify the affected component.
I coordinated with teams, applied mitigation, and restored service quickly.
After recovery, I performed RCA using timeline analysis and 5 Whys.
I implemented monitoring and automation to prevent recurrence.

**4. How do you perform Root Cause Analysis (RCA)?**
   
I start by collecting incident details, timeline, logs, metrics, and recent changes.
I separate symptoms from the actual root cause.
I use techniques like 5 Whys and impact analysis.
I document root cause, resolution, and preventive actions.
The main goal is preventing the same incident from happening again.

**5. Deployment succeeded but production became unhealthy. What will you do?**
    
I will compare application health before and after deployment.
Check logs, error rates, pod status, configuration changes, and dependencies.
Verify database migrations and environment-specific issues.
If customer impact is high, I will rollback to the stable version.
After recovery, I will improve deployment validation and testing.

**6. How do you monitor a critical payment application like Mastercard?**
    
I monitor availability, latency, error rate, throughput, and transaction success rate.
Infrastructure metrics include CPU, memory, disk, network, and Kubernetes health.
Application logs and distributed tracing help identify failures.
Business metrics like successful and failed transactions are very important.
I define SLI, SLO, and alerts based on customer impact.

**7. Kubernetes pod is in CrashLoopBackOff. How will you troubleshoot?**
    
First, I check pod status and events using kubectl describe command.
I review container logs to understand application failures.
Verify configuration, secrets, environment variables, and resource limits.
Check readiness/liveness probes and dependency connectivity.
Fix the issue, redeploy, and monitor pod stability.

**8. How would you improve application availability, scalability, and fault tolerance?**
    
I will use highly available architecture with multiple instances and load balancing.
Implement auto-scaling based on traffic and resource utilization.
Use caching, database optimization, and resilient service design.
Implement health checks, monitoring, and disaster recovery.
Regular testing ensures the application handles failures properly.

**9. Tell me about an automation you implemented that improved operations.**
    
I automated a manual operational process using Jenkins pipeline and scripting.
The automation performed validation, execution, monitoring, and reporting.
It reduced manual effort and minimized human errors.
Added logging and metrics improved visibility.
This helped the team improve reliability and reduce operational toil.


**10. Explain SLI, SLO, and SLA. How do you calculate them?**
    
SLI is the actual measurement of service performance, such as availability or latency.
SLO is the internal reliability target, for example 99.9% availability.
SLA is the agreement with customers/business teams.
Availability SLI = Successful requests / Total requests × 100.
SLO compliance is calculated by comparing actual performance against the defined target.

In my project, we use Prometheus and Grafana to measure service reliability. The main SLI we track is API availability and successful workflow processing. For example, if CMPA processes 99,900 successful requests out of 100,000 total requests, the availability SLI is 99.9%. We define an SLO of 99.9% monthly availability, meaning the service should remain available for at least 99.9% of the time. If the actual availability falls below 99.9%, it is considered an SLO breach. The SLA is the business commitment for service availability and incident response, and we monitor our SLI to ensure we meet the agreed SLO and SLA targets


**11. What is MTTR and how do you reduce it?**
    
MTTR (Mean Time To Recovery/Repair) is the average time taken to restore service after an incident.
Formula: MTTR = Total incident resolution time / Number of incidents.
I reduce MTTR by improving monitoring, faster alerting, automation, and clear runbooks.
I use rollback strategies, self-healing mechanisms, and automated recovery processes.
After incidents, I analyze RCA and implement improvements to prevent delays.

**12. What is MTTD and how do you improve it?**
    
MTTD (Mean Time To Detect) is the average time taken to identify an incident after it starts.
Formula: MTTD = Total time to detect incidents / Number of incidents.
I improve MTTD by implementing effective monitoring, alerting, dashboards, and log analysis.
I configure proactive alerts based on SLI/SLO metrics like availability, latency, and errors.
Better observability helps detect issues early before they impact customers.

**13. MTTA -> mean time to acknowledge it**
