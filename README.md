# AWS EBS Volume Optimization Methodology

> **Note:** This is a generalized, sanitized framework based on real production work that delivered significant annual AWS savings at enterprise scale. It reflects lessons learned and refinements made after execution, not necessarily the exact process used during the original campaign.

---

## Overview

A systematic approach for safely identifying and removing orphaned EBS volumes in production AWS environments, balancing cost optimization with operational safety.

This methodology was developed and validated through a production cleanup campaign across a large-scale EKS environment (1,000+ namespaces, 1,000+ EBS volumes analyzed).

---

## Key Results (from original implementation)

**Direct campaign (original author):**
- 729 volumes deleted
- 50+ TB storage reclaimed
- Significant annual savings
- Zero production incidents

**Adoption by colleague (following the campaign):**
- 3 high-IOPS io1 volumes identified and removed
- Additional significant annual savings

**Combined organizational impact:**
- **732 total volumes removed**
- **Significant combined annual savings**
- **Zero production incidents**

**Timeline:** ~5 weeks (phased execution with monitoring between batches)

---

## The Critical Insight: IOPS Cost Drivers

During the campaign, one discovery changed everything:

**A single 750GB io1 volume with 15,000 provisioned IOPS:**

| Cost Component | % of Total |
|----------------|------------|
| Storage (750GB) | ~7% |
| IOPS (15,000) | ~93% |

**Key lesson:** For io1/io2 volumes, IOPS often dominates cost while being invisible in basic volume listings. Always check provisioned IOPS when evaluating high-value volumes.

This single volume represented a significant portion of the direct campaign savings.

---

## The 7-Step Verification Framework

This framework represents the refined methodology—what you should use, informed by lessons learned during production execution.

### Step 1: Volume Age & IOPS Analysis (Triage)

**Query unattached volumes** that have been in `available` state for 30+ days.

**Capture:**
- Volume IDs
- Size (GB)
- Volume type (gp2, gp3, io1, io2, etc.)
- Age (days since last attachment)
- Tags (owner, environment, service)
- **For io1/io2: Provisioned IOPS** (critical cost factor)

**Output:** List of candidate volumes for further verification, prioritized by age and cost impact.

---

### Step 2: Attachment State Verification

**Verify volume is unattached:**

```bash
aws ec2 describe-volumes --filters Name=status,Values=available
```

- Status `available` means unattached
- But unattached ≠ deletable — proceed to further verification

---

### Step 3: Kubernetes PV/PVC Cross-Reference

**Check for PersistentVolume references:**

```bash
kubectl get pv -o json | jq '.items[] | select(.spec.awsElasticBlockStore.volumeID)'
```

**Check for PersistentVolumeClaim references:**

```bash
kubectl get pvc --all-namespaces | grep <volume-id>
```

- If PV/PVC exists → Workload expects this storage. **Do not delete.**
- If no references found → Proceed to next check.

**Why this matters:** A volume might be unattached at the EC2 level but still referenced by Kubernetes. Deleting it would break the next pod that tries to mount it.

---

### Step 4: CloudTrail Activity Analysis

**Check when volume was last accessed:**

CloudTrail reveals the last `AttachVolume`, `DetachVolume`, or `CreateSnapshot` events.

- Volumes with no activity for 6+ months are strong deletion candidates
- Recent activity means someone's still using it, even if currently detached

---

### Step 5: Snapshot & AMI Pipeline Check

**Check for backup-related indicators:**

- Velero backup tags
- DR/restore workflow identifiers
- Snapshot/AMI dependencies
- Recovery documentation references

Snapshots and AMIs indicate the volume is referenced by automation, launch templates, or image build pipelines. Even if the volume isn't directly backing an AMI (snapshots do that), its existence in an active workflow means deletion could break restore processes or future builds.

If backup-related signals exist → **Verify with team before deletion.**

---

### Step 6: Tag & Naming Convention Analysis

**Check if tags or naming patterns indicate purpose:**

Tags like `kubernetes.io/created-for/pvc/name` or naming patterns like `prometheus-data-*` reveal original purpose. Cross-reference with current cluster state to determine if that workload still exists.

---

### Step 7: Senior Team Validation for High-Value Volumes

**For volumes >1TB or with significant cost impact:**

- Schedule validation session with senior engineers
- Walk through verification evidence for each volume
- Allow independent verification using alternative methods
- Delete only after consensus

**Why this matters:**
- Builds organizational trust
- Creates shared accountability
- Catches blind spots
- Makes the methodology reusable (not dependent on one person's judgment)

---

## Phased Execution

Don't delete everything on day one. Use a phased approach:

**Phase 1:** Start small (10 oldest, smallest volumes). 48-hour monitoring. Zero issues.

**Phase 2:** Incremental batches (15 → 20 → 30 → 50 volumes), 24-48 hour monitoring windows between each.

**Phase 3:** Medium-value volumes (e.g., decommissioned Prometheus volumes).

**Phase 4:** High-value volumes with senior team validation.

**Original campaign pattern:**
- Started with low-value volumes (<10GB)
- Progressed to medium-value volumes
- Finished with high-value volumes (after team validation)
- **Production incidents: 0**

---

## Implementation Guidance

### What to automate (Steps 1-5)
- Volume discovery and filtering (AWS API)
- PV/PVC cross-referencing (kubectl automation)
- CloudTrail activity analysis
- Tag checking and classification
- Cost calculation and prioritization

### What requires human judgment (Steps 6-7)
- High-value volume decisions
- Backup/DR verification when tags are unclear
- Batch sizing and timing
- Validation with stakeholders

### When to stop and ask
- Volume has unclear tags but high cost
- Backup/DR signals are ambiguous
- Volume is very large (>1TB) with no obvious owner
- Team members express uncertainty

---

## Prevention (Stop This from Recurring)

One-off cleanups buy time. Prevention keeps the bill clean:

### Enforce tagging at creation
- Owner (team/individual)
- Service/Application
- Environment (dev/staging/prod)
- Cost center (for chargeback)

### Implement lifecycle policies
- Reclaim policies on PersistentVolumes (Retain vs Delete)
- Automated orphan detection (>30 days unattached)
- Cost visibility dashboards (by team/namespace)

### Use policy-as-code
- Tag enforcement at merge time
- Pre-deployment validation
- Automated compliance reporting

---

## Key Takeaways

✅ **Systematic beats aggressive**
Zero incidents preserved organizational trust for future optimization work.

✅ **IOPS analysis is non-negotiable for io1/io2**
Storage size alone misses 90%+ of cost for high-IOPS volumes.

✅ **Phased execution enables learning**
Early batches revealed patterns that guided later decisions.

✅ **Team validation builds repeatability**
Independent verification meant the methodology could be reused by others.

✅ **Automate what you've validated**
Once you trust the methodology, bake it into your processes. Manual verification doesn't scale.

---

## Limitations & Disclaimers

- This framework is organization-agnostic. Adapt checks to your specific infrastructure patterns.
- Implementation scripts from the original campaign remain internal/proprietary.
- Specific timeframes (monitoring windows, stakeholder response times) will vary by organization.
- **Always prioritize safety over speed. Zero incidents is the goal.**

---

## Resources

**AWS EBS Pricing:**
https://aws.amazon.com/ebs/pricing/

**FinOps Foundation:**
https://www.finops.org/

---

## Contributing

This methodology is shared to help platform engineers run safe, repeatable cost optimization campaigns. If you've used this framework and have improvements or lessons learned, contributions are welcome.

Questions? Open an issue or reach out via GitHub.

---

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

**You are free to:**
- Share and adapt this material for any purpose, including commercial use

**Under the following terms:**
- **Attribution** — Credit the original work and indicate if changes were made

Full license: https://creativecommons.org/licenses/by/4.0/legalcode
