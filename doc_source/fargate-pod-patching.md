# Fargate Pod patching<a name="fargate-pod-patching"></a>

Amazon EKS must periodically patch AWS Fargate Pods to keep them secure\. Updates are attempted in a way that creates the least impact on your services\. However, if Pods aren't successfully evicted, there are times when they must be deleted\. The following are actions that you can take to minimize potential disruptions:
+ Set appropriate Pod disruption budgets \(PDBs\) to control the number of Pods that are down simultaneously\.
+ Create event rules to react to failed evictions before the Pods are deleted\.

Amazon EKS works closely with the Kubernetes community to make bug fixes and security patches available as quickly as possible\. New patch versions of Kubernetes are made available as part of Fargate platform versions\. All Fargate Pods start on the most recent Kubernetes patch version, which is available from Amazon EKS for the Kubernetes version of your cluster\. If you have a Pod with an older patch version, Amazon EKS might restart it to update it to the latest version\. This ensures that your Pods are equipped with the latest security updates\. That way, if there's a critical [Common Vulnerabilities and Exposures](https://cve.mitre.org/) \(CVE\) issue, you're kept up to date to reduce security risks\.

To limit the number of Pods that are down at one time when Pods are patched, you can set Pod disruption budgets \(PDBs\)\. You can use PDBs to define minimum availability based on the requirements of each of your applications while still allowing updates to occur\. For more information, see [Specifying a Disruption Budget for your Application](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) in the *Kubernetes Documentation*\.

Amazon EKS uses the [Eviction API](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#eviction-api) to safely drain the Pod while respecting the PDBs that you set for the application\. Pods are evicted by Availability Zone to minimize impact\. If the eviction succeeds, the new Pod gets the latest patch and no further action is required\.

When the eviction for a Pod fails, Amazon EKS sends an event to your account with details about the Pods that failed eviction\. You can act on the message before the scheduled termination time\. The specific time varies based on the urgency of the patch\. When it's time, Amazon EKS attempts to evict the Pods again\. However, this time a new event isn't sent if the eviction fails\. If the eviction fails again, your existing Pods are deleted periodically so that the new Pods can have the latest patch\.

The following is a sample event received when the Pod eviction fails\. It contains details about the cluster, Pod name, Pod namespace, Fargate profile, and the scheduled termination time\.

```
{
    "version": "0",
    "id": "12345678-90ab-cdef-0123-4567890abcde",
    "detail-type": "EKS Fargate Pod Scheduled Termination",
    "source": "aws.eks",
    "account": "111122223333",
    "time": "2021-06-27T12:52:44Z",
    "region": "region-code",
    "resources": [
        "default/my-database-deployment"
    ],
    "detail": {
        "clusterName": "my-cluster",
        "fargateProfileName": "my-fargate-profile",
        "podName": "my-pod-name",
        "podNamespace": "default",
        "evictErrorMessage": "Cannot evict Pod as it would violate the Pod's disruption budget",
        "scheduledTerminationTime": "2021-06-30T12:52:44.832Z[UTC]"
    }
}
```

In addition, having multiple PDBs associated with a Pod can cause an eviction failure event\. This event returns the following error message\.

```
"evictErrorMessage": "This Pod has multiple PodDisruptionBudget, which the eviction subresource does not support",
```

You can create a desired action based on this event\. For example, you can adjust your Pod disruption budget \(PDB\) to control how the Pods are evicted\. More specifically, suppose that you start with a PDB that specifies the target percentage of Pods that are available\. Before your Pods are force terminated during an upgrade, you can adjust the PDB to a different percentage of Pods\. To receive this event, you must create an Amazon EventBridge rule in the AWS account and Region that the cluster belongs to\. The rule must use the following **Custom pattern**\. For more information, see [Creating Amazon EventBridge rules that react to events](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html) in the *Amazon EventBridge User Guide*\.

```
{
  "source": ["aws.eks"],
  "detail-type": ["EKS Fargate Pod Scheduled Termination"]
}
```

A suitable target can be set for the event to capture it\. For a complete list of available targets, see [Amazon EventBridge targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html) in the *Amazon EventBridge User Guide*\.