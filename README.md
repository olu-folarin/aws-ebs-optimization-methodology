# AWS EBS Volume Optimization Methodology

> **Portfolio Note**: This repository documents the systematic methodology I created and successfully applied in production. For the actual implementation (shell scripts, campaign report, and 729 deletion execution details), see the [finops-ebs-cleanup repository](https://github.com/FolarinOyenuga/finops-ebs-cleanup).
>
> **Blog Post**: [How Systematic EBS Optimization Delivered £108K in Annual Savings](link-when-published)

A systematic 7-step verification framework for safely identifying and removing orphaned EBS volumes in production AWS environments, preventing false positives while enabling confident cost optimization at scale.

## Impact

### Real-World Production Results

When I applied this methodology in a UK government production environment managing 1,000+ EBS volumes:

**My Direct Campaign (live-1 cluster)**:
- **£62,829 annual savings** (verified via AWS Cost Explorer)
- **729 volumes deleted** with 100% success rate (0 incidents)
- **48.65 TB storage reclaimed**
- **12 phased batches** over 5 weeks (Oct-Nov 2025)

**Methodology Adoption (live-2 cluster)**:
- Colleague applied this framework to identify over-provisioned io1 volumes
- **£45,241 additional annual savings** from just 3 volumes
- Demonstrates force multiplication through reusable methodology

**Combined Organizational Impact**: **£108,070 annually**

**Actual Implementation**: See [finops-ebs-cleanup](https://github.com/FolarinOyenuga/finops-ebs-cleanup) for complete scripts, campaign report, and execution details *(work GitHub account)*.

## The Problem

Orphaned EBS volumes represent significant cloud waste—resources that persist after instances are terminated or Kubernetes PVCs are deleted. However, rushing to delete volumes without thorough verification risks production outages. Standard approaches often result in:

- **False positives** - deleting volumes still referenced by disaster recovery systems
- **Service disruption** - removing volumes needed for blue/green deployments
- **Data loss** - eliminating volumes without confirmed backup coverage
- **Compliance violations** - deleting resources without stakeholder approval

## The Solution: 7-Step Verification Framework

This methodology balances aggressive cost optimization with operational safety through systematic validation at each step.

### Step 1: Identify Candidate Volumes

Query for volumes meeting orphan criteria:
- **Status:** `available` (unattached)
- **Age threshold:** 30+ days unattached
- **Export data:** volume-id, size, age, last-attached-instance, tags

**Why this matters:** Recent volumes (<30 days) may be temporarily unattached during deployments or testing.
```bash
# Example AWS CLI query
aws ec2 describe-volumes \
  --region eu-west-2 \
  --filters "Name=status,Values=available" \
  --query 'Volumes[?CreateTime<=`2024-10-01`].[VolumeId,Size,CreateTime,Tags]' \
  --output table
```

### Step 2: Verify Kubernetes PersistentVolume Status

Check if a PersistentVolume (PV) exists referencing this volume:
```bash
# Extract PV name from volume tags or description
kubectl get pv  --all-namespaces
```

**Decision logic:**
- ✅ **"NotFound"** → Likely orphaned, proceed to Step 3
- ❌ **"Bound/Available/Released"** → Investigate further, DO NOT delete

**Why this matters:** Volumes may have been detached but PVs could still exist with "Released" reclaim policy, indicating intentional preservation.

### Step 3: Confirm PersistentVolumeClaim Absence

Verify no PersistentVolumeClaim (PVC) exists across all namespaces:
```bash
# Search all namespaces for PVC
kubectl get pvc --all-namespaces | grep 

# Alternative: Direct query
kubectl get pvc  -n 
```

**Decision logic:**
- ✅ **No PVC found** → Proceed to Step 4
- ❌ **PVC exists** → Volume is in use, DO NOT delete

**Why this matters:** Even if a volume is unattached, an active PVC indicates the application expects to reattach it.

### Step 4: Check Disaster Recovery Tags

Verify the volume isn't part of backup infrastructure:
```bash
# Check for Velero or backup-related tags
aws ec2 describe-volumes \
  --volume-ids  \
  --query 'Volumes[*].Tags[?Key==`velero.io/backup`]'
```

**Critical tags to check:**
- `velero.io/backup` 
- `snapshot-policy` 
- `backup-retention` 
- `disaster-recovery` 

**Decision logic:**
- ✅ **No backup tags** → Safe to evaluate for deletion
- ❌ **Has backup tags** → Consult DR team before proceeding

**Why this matters:** Backup systems may create volumes that appear orphaned but are essential for recovery operations.

### Step 5: Verify Snapshot Coverage

Before deletion, confirm recovery capability exists:
```bash
# Check for existing snapshots
aws ec2 describe-snapshots \
  --filters "Name=volume-id,Values=" \
  --query 'Snapshots[*].[SnapshotId,StartTime,State]'
```

**Decision logic:**
- ✅ **Recent snapshots exist** → Deletion is recoverable
- ⚠️ **No snapshots** → Create snapshot before deletion
- ❌ **Critical data, no snapshots** → Investigate why backups don't exist

**Why this matters:** Snapshots provide a recovery path if deletion was incorrect. For volumes >1TB, snapshot creation can take hours—plan accordingly.

### Step 6: Stakeholder Consultation

Notify namespace/resource owners before deletion:
- **Communication channel:** Slack, email, or ticketing system
- **Response window:** 48-72 hours
- **Required information:**
  - Volume ID, size, age
  - Last attached instance
  - Namespace/team ownership
  - Planned deletion date

**Template message:**
```
Volume ID: vol-xxxxx
Size: 100GB
Unattached since: 2024-06-01 (218 days)
Last instance: i-xxxxx (terminated)
Namespace: monitoring

This volume appears orphaned. We plan to delete it on [DATE]. 
If you need it preserved, please respond within 48 hours.
Snapshot available: snap-xxxxx
```

**Why this matters:** Teams may have legitimate reasons for keeping unattached volumes (compliance, auditing, disaster recovery scenarios). Consultation prevents surprise disruptions.

### Step 7: Batch Deletion with Monitoring Windows

Delete in controlled batches with observation periods:

**Batch strategy:**
1. **First batch:** 5-10 smallest volumes (<10GB)
2. **Monitoring window:** 24-48 hours
3. **Validation checks:**
   - No increase in support tickets
   - No application errors in logs
   - No unexpected PVC creation attempts
4. **Repeat:** Incrementally larger batches

**Deletion command:**
```bash
# Delete with confirmation
aws ec2 delete-volume --volume-id 

# Verify deletion
aws ec2 describe-volumes --volume-ids 
# Expected: "InvalidVolume.NotFound" error
```

**Why this matters:** Batch approach enables rapid rollback if issues arise. Small initial batch limits blast radius if verification missed an edge case.

## Cost Calculation Methodology

Calculate savings using AWS pricing model:
```python
# EBS gp3 pricing (eu-west-2): £0.088/GB/month

monthly_savings = total_gb * 0.088
annual_savings = monthly_savings * 12

# Example: 732 volumes, 50.9TB
monthly_savings = 52,121 GB * £0.088 = £4,586.65
annual_savings = £55,039.80

# Add IOPS optimization savings for io1/io2 volumes
iops_monthly = provisioned_iops * £0.071
```

**Document all calculations** to enable verification by finance/leadership teams.

## Cost Driver Analysis: The £15K io1 Volume

During my campaign, one volume stood out as the single largest cost contributor:

**750GB io1 volume with 19,500 provisioned IOPS**

**Annual cost breakdown**:
- IOPS cost: £14,067 (93% of total)
- Storage cost: £1,013 (7% of total)
- **Total annual cost**: £15,080

**Key insight**: The IOPS cost was **13.6× more expensive** than the storage itself.

This single orphaned volume represented **24% of my entire campaign savings**. It had been provisioned for high-throughput workloads that no longer existed, but the IOPS allocation persisted, silently draining budget.

**Lesson**: Always check io1/io2 IOPS utilization. Over-provisioned IOPS is often the most expensive form of cloud waste—and the easiest to overlook during cleanup campaigns.

This discovery directly led to the live-2 cluster optimization where my colleague found 3 similar over-provisioned volumes worth £45K annually.

## Methodology Adoption

**live-2 Cluster Optimization** (Colleague):
- Applied this 7-step framework to identify over-provisioned volumes
- Found 3×750GB io1 volumes with excessive IOPS (19,500 each)
- **Result**: £45,241 annual savings from just 3 volumes

**Modernisation Platform Team**:
- Built automation tooling based on this verification framework
- Now using as standard operating procedure for EBS cleanup
- Specific savings from their implementation not yet quantified

This demonstrates the methodology's reusability—others can apply it to their AWS environments without my direct involvement.

**Key success factors for adoption:**
1. **Clear documentation** - Each step explained with rationale
2. **Safety-first approach** - Zero production incidents build confidence
3. **Quantified results** - £108K savings provide compelling business case
4. **Reusable process** - Not team-specific, works across AWS environments

## Lessons Learned

**What worked well:**
- Batch deletion with monitoring prevented any false positives from escalating
- 7-step verification caught multiple edge cases (Velero volumes, blue/green ENIs)
- Stakeholder consultation prevented 3 premature deletions

**What to improve:**
- **Automation opportunity:** Steps 1-5 can be automated with Lambda functions
- **Retention policies:** Prevent orphan creation through lifecycle management
- **Proactive monitoring:** Alert when volumes remain unattached >14 days

## Future Enhancements

**Automation roadmap:**
1. **Detection pipeline:** Lambda + EventBridge to identify candidates daily
2. **Automated validation:** Kubernetes API integration for PV/PVC checks
3. **Dashboard:** Grafana visualization of orphan volumes over time
4. **Alerting:** Notify teams when volumes approach orphan threshold

**Preventive measures:**
1. **Reclaim policies:** Ensure PVs use `Delete` reclaim policy where appropriate
2. **Lifecycle rules:** Automatic snapshot creation before volume deletion
3. **Tagging standards:** Mandatory ownership tags for all volumes

## Related Methodologies

- **EBS Snapshot Optimization:** Similar framework for orphaned snapshots
- **ENI Cleanup Validation:** How to verify network interfaces before deletion
- **RDS Volume Right-sizing:** Methodology for database storage optimization

## License

MIT License - Use freely, attribution appreciated.

---

## Related Resources

**Actual Implementation**: [finops-ebs-cleanup](https://github.com/FolarinOyenuga/finops-ebs-cleanup) - Complete shell scripts, phased deletion batches, and detailed campaign report *(work account)*

**Blog Post**: [How Systematic EBS Optimization Delivered £108K](link-when-published) - Full story with lessons learned

**Author**: Folarin Oyenuga  
**LinkedIn**: [folarin-o-46389b128](https://www.linkedin.com/in/folarin-o-46389b128/)  
**GitHub**: [github.com/olu-folarin](https://github.com/olu-folarin)  
**Email**: folarinoyenuga200@gmail.com

---

**Note**: This methodology repository lives on my personal GitHub. The actual production implementation (with scripts and campaign report) is in my work GitHub account as it was employer work. Both are publicly available.
