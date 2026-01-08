# INTERVIEW DAY QUICK REFERENCE CHEAT SHEET
## Sri Ram Mogalapalli - Senior DevOps Engineer

---

## 📊 KEY METRICS (MEMORIZE THESE)

### GitHub Actions Pipeline
- **Time:** 4h → 30min (87.5% ↓)
- **Frequency:** 2/week → 10/week (400% ↑)
- **Success:** 99.2%
- **Rollbacks:** 85% ↓
- **Defects caught:** 92% pre-prod

### Serverless Migration
- **Cost:** $120K/year saved (45% ↓)
- **Scale:** 10M+ req/day
- **Latency:** <200ms p95
- **Uptime:** 99.9%

### Cost Optimization
- **Total:** $283K/year saved
- **Serverless:** $120K
- **Optimization:** $180-200K

### Incident Management
- **MTTR:** 4h → 45min (70% ↓)
- **Recurrence:** 80% ↓
- **Self-healing:** 70% manual fixes eliminated

### Team Impact
- **Trained:** 25+ engineers
- **Onboarding:** 50% faster
- **Certifications:** 80%
- **Promotions:** 100% in <18mo

---

## 🎯 DRISHYA ACHIEVEMENTS (30-SEC VERSION)

"Over 4.5 years at Drishya AI Labs, I:
- Reduced infrastructure costs by $300K+ annually through serverless migration and optimization
- Built GitHub Actions CI/CD reducing deploy time from 4 hours to 30 minutes
- Led SOC 2 Type 2 certification in 4 months
- Migrated AWS to Azure in 18 days with zero downtime
- Managed 15+ AWS accounts and Kubernetes clusters
- Mentored 25+ engineers with 100% promotion rate"

---

## 🔧 TECHNICAL STACK (ONE-LINERS)

### AWS
**Lambda:** Serverless compute, optimized cold starts <200ms, provisioned concurrency for critical paths
**ECS/Fargate:** Container orchestration, deployed MLflow with private VPC + ALB
**VPC:** Multi-AZ design, NAT gateways, VPC endpoints for cost savings
**IAM:** Multi-account SSO, cross-account roles, least privilege policies

### Azure
**AKS:** Managed Kubernetes, node pools, Azure CNI networking, AAD integration
**Networking:** VNets, NSGs, Application Gateway for ingress

### Terraform
**State:** S3 backend + DynamoDB locking, versioning enabled
**Modules:** Reusable for VPC, EKS, RDS, 15+ AWS accounts
**Best Practice:** Separate state per environment, workspaces for variants

### Kubernetes
**Scheduling:** Resource requests/limits, node affinity, taints/tolerations
**Scaling:** HPA (CPU/memory), Cluster Autoscaler
**Security:** RBAC, Pod Security Standards, Network Policies
**Troubleshooting:** CrashLoopBackOff, ImagePullBackOff, OOMKilled

### CI/CD
**GitHub Actions:** Matrix builds, reusable workflows, OIDC auth, self-hosted runners
**Pipeline:** Lint → Test → Build → Deploy, quality gates, auto-rollback
**Container:** Multi-stage Dockerfile, layer caching, registry sync (AWS/Azure)

### Monitoring
**Prometheus:** PromQL, alerting rules, 15-sec scrape interval
**Grafana:** Real-time dashboards, template variables, PagerDuty integration
**CloudWatch:** Metrics, Logs Insights, alarms, custom metrics

---

## 💬 COMMON QUESTIONS - QUICK ANSWERS

### "Tell me about yourself" (90 seconds)
"I'm a Senior DevOps Engineer with 4.5 years of experience at Drishya AI Labs, specializing in multi-cloud infrastructure, CI/CD automation, and cost optimization. I've reduced deployment time by 87% through GitHub Actions pipelines, saved $300K+ annually via serverless migration and infrastructure optimization, and led SOC 2 Type 2 certification. I manage 15+ AWS accounts, Kubernetes clusters with 15+ services, and have mentored 25+ engineers. My background combines electrical engineering education with hands-on DevOps, MLOps, and cloud architecture. I'm passionate about building scalable, cost-efficient systems that enable teams to ship faster."

### "Why are you looking for a new role?"
"I've accomplished a lot at Drishya—automated deployments, optimized costs, achieved compliance certification—but I'm ready for new challenges at scale. I'm looking for opportunities to work on larger, more complex distributed systems, contribute to open-source DevOps tools, and mentor larger teams. I'm particularly interested in companies pushing the boundaries of cloud-native architectures and infrastructure automation."

### "What's your biggest strength?"
"Problem-solving through data. When faced with challenges, I analyze metrics, identify root causes, and implement solutions. For example, our 4-hour deployments—I measured each stage, found the bottlenecks (testing, builds, manual steps), and systematically addressed each with parallelization, caching, and automation, resulting in 87% time reduction. I combine technical depth with business understanding to deliver measurable results."

### "What's your biggest weakness?"
"Earlier in my career, I'd jump into implementation before fully understanding requirements. I once rebuilt a monitoring dashboard that users didn't need because I didn't ask enough questions. Now I always start with 'why'—understanding the business problem before proposing technical solutions. This has made my work more impactful and reduced wasted effort."

### "Tell me about a time you failed"
"During a Kubernetes cluster upgrade, I caused a 2-hour production outage because I underestimated the complexity. I tested in staging but didn't account for deprecated API versions and different resource limits in production. I had to rollback, delaying the upgrade by a week. The lesson: I now create production-like staging environments, use pre-upgrade API scanning tools, and always have tested rollback procedures. Every subsequent upgrade has been zero-downtime because of these lessons."

### "Where do you see yourself in 5 years?"
"I see myself as a Staff/Principal DevOps Engineer or DevOps Engineering Manager, leading platform teams that enable hundreds of engineers to ship reliably. I want to define best practices, contribute to open-source infrastructure tools, and mentor the next generation of DevOps engineers. I'm also interested in the intersection of DevOps and AI/ML infrastructure—making ML models as deployable and scalable as traditional applications."

---

## 🏗️ SYSTEM DESIGN FRAMEWORK (5-STEP)

### 1. Requirements (5 min)
```
Functional:
- What does it do? Core features?
- APIs/interfaces?

Non-Functional:
- Scale (DAU, QPS, data size)
- Performance (latency, throughput)
- Availability (SLA)
- Consistency (strong vs eventual)
```

### 2. Estimation (5 min)
```
- Storage: X users × Y bytes = Z TB
- Bandwidth: Z req/sec × W bytes = Q Mbps
- QPS: Daily active / 86400 × peak multiplier
- Cache: 20% of hot data
```

### 3. High-Level Design (10 min)
```
Client → CDN → Load Balancer → App Servers → Database
         ↓
       Cache
         ↓
      Queue → Workers
```

### 4. Deep Dive (15 min)
```
- Database schema
- API contracts
- Caching strategy
- Scaling approach
- Failure scenarios
```

### 5. Trade-offs (5 min)
```
- Pros/cons of choices
- Alternative approaches
- 10x scale considerations
```

---

## 📚 KUBERNETES TROUBLESHOOTING CHEAT SHEET

```bash
# Pod stuck in Pending
kubectl describe pod <pod> | grep -A 5 Events
# Check: Insufficient resources, node selectors, PVC not bound

# CrashLoopBackOff
kubectl logs <pod> --previous  # Check previous container logs
# Common: OOMKilled (increase limits), missing env vars, app crash

# ImagePullBackOff
kubectl describe pod <pod> | grep Image
# Check: Image name typo, registry auth, network policy

# Service not reachable
kubectl get endpoints <service>  # Should show pod IPs
# Check: Label selector mismatch, pod not ready, network policy

# High memory usage
kubectl top pods
kubectl exec <pod> -- ps aux  # Find memory-heavy process
# Solution: Increase limits, optimize app, HPA

# Node NotReady
kubectl describe node <node>
# Check: Disk pressure, memory pressure, kubelet issues
```

---

## 💰 COST OPTIMIZATION PLAYBOOK

### Compute
- [ ] Right-size instances (use actual metrics)
- [ ] Spot instances for batch workloads (75% savings)
- [ ] Serverless for variable workloads
- [ ] Auto-scaling policies
- [ ] Schedule stop/start for non-prod

### Storage
- [ ] S3 lifecycle policies (Standard → IA → Glacier)
- [ ] Delete unused EBS volumes/snapshots
- [ ] Enable compression
- [ ] Archive old logs

### Networking
- [ ] CloudFront/CDN for static assets
- [ ] VPC endpoints (avoid NAT Gateway $)
- [ ] Compress data transfer
- [ ] Use S3 Transfer Acceleration wisely

### Monitoring
- [ ] CloudWatch Logs retention policies
- [ ] Filter metrics (don't send everything)
- [ ] Use cost anomaly detection
- [ ] Budget alerts at 80%, 90%, 100%

---

## 🔐 SOC 2 COMPLIANCE QUICK FACTS

**What is SOC 2?**
- Security, Availability, Processing Integrity, Confidentiality, Privacy
- Type I: Point-in-time audit
- Type II: 3-12 month operational audit (what we achieved)

**Key Controls Implemented:**
- Access control: IAM policies, MFA, SSO
- Encryption: At rest (KMS) and in transit (TLS)
- Logging: CloudTrail, audit logs, retention
- Change management: GitHub PR reviews, approval workflows
- Incident response: Runbooks, PagerDuty, RCA process
- Vulnerability management: Automated scanning, patching

**Timeline:** 4 months from kickoff to certification

---

## 🎤 BEHAVIORAL QUESTION FRAMEWORK (STAR)

**Situation:** (20 sec)
- Set context: Company, team, challenge
- Quantify problem if possible

**Task:** (10 sec)
- Your specific responsibility
- Goals/constraints

**Action:** (60 sec)
- What YOU did (not "we")
- Step-by-step approach
- Decisions you made
- Challenges faced

**Result:** (20 sec)
- Quantifiable outcomes
- Business impact
- Lessons learned

**Total:** 2-3 minutes max per story

---

## ❓ QUESTIONS FOR INTERVIEWER

### Technical/Role
1. "What are the biggest infrastructure challenges you're facing?"
2. "How mature are your DevOps practices? (CI/CD, IaC, monitoring)"
3. "What's the on-call rotation like?"
4. "Tell me about a recent complex project the team worked on."

### Team/Culture
5. "How does DevOps collaborate with product/engineering?"
6. "What does success look like in this role in 6 months?"
7. "How does the team approach learning and skill development?"
8. "Can you describe the team structure and dynamics?"

### Growth
9. "What are the career growth opportunities?"
10. "How does the company support professional development?"
11. "Are there opportunities to work with cutting-edge technologies?"

---

## 🧠 CONFIDENCE REMINDERS

**You have:**
- ✅ 4.5 years of solid DevOps experience
- ✅ $300K+ proven cost savings
- ✅ Multi-cloud expertise (AWS, Azure, GCP)
- ✅ Real production impact (99.9% uptime, 10M+ requests/day)
- ✅ Leadership experience (mentored 25+ engineers)
- ✅ Compliance knowledge (SOC 2 Type 2)

**Remember:**
- Take time to think before answering
- Ask clarifying questions
- Be honest if you don't know something
- Show enthusiasm and curiosity
- It's okay to say "Let me walk through my thought process"

**You've got this!** 🚀

---

## 📱 LAST-MINUTE CHECKLIST

**Technical Setup:**
- [ ] Quiet space, good lighting
- [ ] Webcam/mic tested
- [ ] Backup internet (hotspot)
- [ ] Phone silenced
- [ ] Whiteboard/paper ready

**Materials:**
- [ ] Resume printed (2 copies)
- [ ] This cheat sheet
- [ ] Questions for interviewer
- [ ] Pen and notepad

**Mental:**
- [ ] Good night's sleep
- [ ] Reviewed STAR stories
- [ ] Practiced key metrics
- [ ] Positive mindset

**During Interview:**
- Think out loud
- Ask clarifying questions
- Use STAR for behavioral
- Draw diagrams
- Show enthusiasm
- Be yourself

---

**Good luck, Sri Ram!**
