# AWS EBS Volume Optimization Methodology

> **Note:** This is a generalized, sanitized framework based on real production work that delivered £108K in annual AWS savings at UK Government scale. It reflects lessons learned and refinements made after execution, not necessarily the exact process used during the original campaign.
>
> **Original work documented here:** https://github.com/ministryofjustice/cloud-platform/issues/7662

---

## Overview

A systematic approach for safely identifying and removing orphaned EBS volumes in production AWS environments, balancing cost optimization with operational safety.

This methodology was developed and validated through a production cleanup campaign across a large-scale EKS environment (200+ namespaces, 1,289 EBS volumes analyzed).

---

## Key Results (from original implementation)

**Direct campaign (original author):**
- 729 volumes deleted
- 50.9TB storage reclaimed
- **£62,829 annual savings**
- Zero production incidents

**Adoption by colleague (following the campaign):**
- 3 high-IOPS io1 volumes identified and removed
- **£45,241 additional annual savings**

**Combined organizational impact:**
- **732 total volumes removed**
- **£108,070 annual savings**

**Timeline:** ~2.5 weeks (phased execution with monitoring between batches)

---

## The Critical Insight: IOPS Cost Drivers

During the campaign, one discovery changed everything:

**A single 750GB io1 volume with 19,500 provisioned IOPS:**

| Cost Component | Annual Cost | % of Total |
|---|---:|---:|
| Storage (750GB) | £1,044 | 7% |
| IOPS (19,500) | £14,040 | 93% |
| **Total** | **£15,084** | **100%** |

**Key lesson:** For io1/io2 volumes, IOPS often dominates cost while being invisible in basic volume listings. Always check provisioned IOPS when evaluating high-value volumes.

This single volume represented 24% of the direct campaign savings.

---

## The 7-Step Verification Framework

This framework represents the refined methodology—what you should use, informed by lessons learned during production execution.

### Step 1: Candidate Identification (AWS)

**Query unattached volumes** that have been in `available` state for 30+ days.

**Capture:**
- Volume IDs
- Size (GB)
- Volume type (gp2, gp3, io1, io2, etc.)
- Age (days since last attachment)
- Tags (owner, environment, service)
- **For io1/io2: Provisioned IOPS** (critical cost factor)

**Output:** List of candidate volumes for further verification.

---

### Step 2: Kubernetes PersistentVolume Check

**Verify no PV references exist:**
```bash
kubectl get pv <pv-name>
```

- If returns "NotFound" → No Kubernetes infrastructure binding exists
- If PV exists (even if "Released") → **Stop and investigate.** The cluster still knows about this volume.

**Why this matters:** Prevents deleting volumes that workloads might expect on next deployment.

---

### Step 3: PersistentVolumeClaim Verification

**Check for active claims across all namespaces:**
```bash
kubectl get pvc --all-namespaces | grep <volume-id>
```

- If PVC exists → Workload expects this storage. **Do not delete.**
- If no PVC found → Proceed to next check.

---

### Step 4: Backup/Disaster Recovery Tag Assessment

**Check for backup-related tags:**
- Velero backup tags
- DR/restore workflow identifiers
- Snapshot/AMI dependencies
- Recovery documentation references

If backup-related tags exist → **Verify with team before deletion.** These volumes often look abandoned but are critical for recovery.

---

### Step 5: Age and Cost Prioritization

**Prioritize candidates by:**
- Age (older = lower risk)
- Cost (higher = higher impact)
- Volume type (io1/io2 = check IOPS, gp2/gp3 = focus on size)

**For high-value volumes:**
- Conduct additional validation
- Document decision rationale
- Get team confirmation before deletion

---

### Step 6: Team Validation for High-Value Volumes

**For volumes with significant cost or size:**
- Schedule validation session with senior engineers
- Walk through verification evidence
- Allow independent verification using alternative methods
- Delete only after consensus

**Why this matters:**
- Builds organizational trust
- Creates shared accountability
- Catches blind spots
- Makes the methodology reusable (not dependent on one person's judgment)

---

### Step 7: Phased Deletion with Monitoring

**Execute in controlled batches:**
1. **Start small:** Delete 5-10 of the oldest, smallest volumes
2. **Monitor:** Observe for issues (deployments, alerts, team reports)
3. **Scale gradually:** Increase batch size as confidence grows
4. **Continue monitoring:** Watch between batches for unexpected impacts

**Original campaign pattern:**
- Started with low-value volumes (<10GB)
- Progressed to medium-value volumes (75GB Prometheus sets)
- Finished with high-value volumes (after team validation)
- **Production incidents: 0**

---

## Implementation Guidance

### What to automate (Steps 1-4)
- Volume discovery and filtering (AWS API)
- PV/PVC cross-referencing (kubectl automation)
- Tag checking and classification
- Cost calculation and prioritization

### What requires human judgment (Steps 5-7)
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
- Tag enforcement at merge time (e.g., terraform-tag-validator)
- Pre-deployment validation
- Automated compliance reporting

---

## Success Factors

From the original campaign:

✅ **Systematic approach beats aggressive deletion**
Zero incidents preserved organizational trust for future optimization work.

✅ **IOPS analysis is non-negotiable for io1/io2**
Storage size alone misses 90%+ of cost for high-IOPS volumes.

✅ **Phased execution enables learning**
Early batches revealed patterns (decommissioned Prometheus volumes, IOPS over-provisioning) that guided later decisions.

✅ **Team validation builds repeatability**
Independent verification meant the methodology could be reused by others (proven by colleague's £45K savings).

---

## Limitations & Disclaimers

- This framework is organization-agnostic. Adapt checks to your specific infrastructure patterns.
- Implementation scripts from the original campaign remain internal/proprietary.
- Specific timeframes (monitoring windows, stakeholder response times) will vary by organization.
- **Always prioritize safety over speed. Zero incidents is the goal.**

---

## Resources

**Original production work:**
https://github.com/ministryofjustice/cloud-platform/issues/7662

**AWS EBS Pricing:**
https://aws.amazon.com/ebs/pricing/

**FinOps Foundation:**
https://www.finops.org/

---

## Contributing

This methodology is shared to help platform engineers run safe, repeatable cost optimization campaigns. If you've used this framework and have improvements or lessons learned, contributions are welcome.

Questions? Open an issue or reach out via GitHub.

---

**License:** [CC-BY-4.0](LICENSE) (adapt freely, attribution required)
