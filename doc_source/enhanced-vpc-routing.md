# Enhanced VPC routing in Amazon Redshift<a name="enhanced-vpc-routing"></a>

When you use Amazon Redshift enhanced VPC routing, Amazon Redshift forces all [COPY](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html) and [UNLOAD](https://docs.aws.amazon.com/redshift/latest/dg/r_UNLOAD.html) traffic between your cluster and your data repositories through your virtual private cloud \(VPC\) based on the Amazon VPC service\. By using enhanced VPC routing, you can use standard VPC features, such as [VPC security groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html), [network access control lists \(ACLs\)](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_ACLs.html), [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints-s3.html), [VPC endpoint policies](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints-s3.html#vpc-endpoints-policies-s3), [internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html), and [Domain Name System \(DNS\)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html) servers, as described in the *Amazon VPC User Guide\.* You use these features to tightly manage the flow of data between your Amazon Redshift cluster and other resources\. When you use enhanced VPC routing to route traffic through your VPC, you can also use [VPC flow logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html) to monitor COPY and UNLOAD traffic\.

 Amazon Redshift clusters and Amazon Redshift Serverless workgroups support enhanced VPC routing\. You can't use enhanced VPC routing with Redshift Spectrum\. For more information, see [Redshift Spectrum and enhanced VPC routing](spectrum-enhanced-vpc.md)\.

If enhanced VPC routing is not turned on, Amazon Redshift routes traffic through the internet, including traffic to other services within the AWS network\.

**Important**  
Because enhanced VPC routing affects the way that Amazon Redshift accesses other resources, COPY and UNLOAD commands might fail unless you configure your VPC correctly\. You must specifically create a network path between your cluster's VPC and your data resources, as described following\.

When you run a COPY or UNLOAD command on a cluster with enhanced VPC routing turned on, your VPC routes the traffic to the specified resource using the *strictest*, or most specific, network path available\. 

For example, you can configure the following pathways in your VPC:
+ ** VPC endpoints **– For traffic to an Amazon S3 bucket in the same AWS Region as your cluster, you can create a VPC endpoint to direct traffic directly to the bucket\. When you use VPC endpoints, you can attach an endpoint policy to manage access to Amazon S3\. For more information about using endpoints with Amazon Redshift, see [Working with VPC endpoints](enhanced-vpc-working-with-endpoints.md)\. If you use Lake Formation, you can find more information about establishing a private connection between your VPC and AWS Lake Formation at [AWS Lake Formation and interface VPC endpoints \(AWS PrivateLink\)](https://docs.aws.amazon.com/lake-formation/latest/dg/privatelink.html)\.
+ **NAT gateway** – You can connect to an Amazon S3 bucket in another AWS Region, and you can connect to another service within the AWS network\. You can also access a host instance outside the AWS network\. To do so, configure a [ network address translation \(NAT\) gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html), as described in the *Amazon VPC User Guide\.*
+ **Internet gateway** – To connect to AWS services outside your VPC, you can attach an [internet gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) to your VPC subnet, as described in the *Amazon VPC User Guide\.* To use an internet gateway, your cluster must have a public IP to allow other services to communicate with your cluster\.

For more information, see [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html) in the Amazon VPC User Guide\.

There is no additional charge for using enhanced VPC routing\. You might incur additional data transfer charges for certain operations\. These include such operations as UNLOAD to Amazon S3 in a different AWS Region\. COPY from Amazon EMR, or Secure Shell \(SSH\) with public IP addresses\. For more information about pricing, see [Amazon EC2 Pricing](https://aws.amazon.com/ec2/pricing/)\.

**Topics**
+ [Working with VPC endpoints](enhanced-vpc-working-with-endpoints.md)
+ [Enhanced VPC routing](enhanced-vpc-enabling-cluster.md)
+ [Redshift Spectrum and enhanced VPC routing](spectrum-enhanced-vpc.md)