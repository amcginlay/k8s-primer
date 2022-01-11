---
title: "Verify CloudWatch Container Insights is working"
chapter: false
weight: 6
---

To verify that data is being collected in CloudWatch, launch the CloudWatch Containers UI in your browser using the link generated by the command below

```bash
echo "
Use the URL below to access Cloudwatch Container Insights in $AWS_REGION:

https://console.aws.amazon.com/cloudwatch/home?region=${AWS_REGION}#cw:dashboard=Container;context=~(clusters~'eksworkshop-eksctl~dimensions~(~)~performanceType~'Service)"
```

![Insights](/images/ekscwci/insights.png)

From here you can see the metrics are being collected and presented to CloudWatch. You can switch between various drop downs to see EKS Services, EKS Cluster and more.

![Metrics Option](/images/ekscwci/metricsoptions.png)

#### We can now continue with load testing the cluster to see how these metrics can look under load.