# Changing cluster encryption<a name="changing-cluster-encryption"></a>

You can modify an unencrypted cluster to use AWS Key Management Service \(AWS KMS\) encryption, using either an AWS\-managed key or a customer managed key\. When you modify your cluster to enable AWS KMS encryption, Amazon Redshift automatically migrates your data to a new encrypted cluster\. You can also migrate an unencrypted cluster to an encrypted cluster by modifying the cluster\. 

During the migration operation, your cluster is available in read\-only mode, and the cluster status appears as **resizing**\. 

If your cluster is configured to enable cross\-AWS Region snapshot copy, you must disable it before changing encryption\. For more information, see [Copying snapshots to another AWS Region](working-with-snapshots.md#cross-region-snapshot-copy) and [Configure cross\-Region snapshot copy for an AWS KMS–encrypted cluster](managing-snapshots-console.md#xregioncopy-kms-encrypted-snapshot)\. You can't enable hardware security module \(HSM\) encryption by modifying the cluster\. Instead, create a new, HSM\-encrypted cluster and migrate your data to the new cluster\. For more information, see [Migrating to an HSM\-encrypted cluster](#migrating-to-an-encrypted-cluster)\. 

**To modify database encryption on a cluster**

1. Sign in to the AWS Management Console and open the Amazon Redshift console at [https://console\.aws\.amazon\.com/redshift/](https://console.aws.amazon.com/redshift/)\.

1. On the navigation menu, choose **Clusters**, then choose the cluster that you want to modify encryption\.

1. Choose **Properties**\.

1. In the **Database configurations** section, choose **Edit**, then choose **Edit encryption**\. 

1. Choose one of the encryption options and choose **Save changes**\.

<a name="changing-cluster-encryption-cli"></a>**To change cluster encryption using the CLI** 

To modify your unencrypted cluster to use AWS KMS, run the `modify-cluster` CLI command and specify `–-encrypted`, as shown following\. By default, your default KMS key is used\. To specify a customer managed key, include the `--kms-key-id` option\.

```
aws redshift modify-cluster --cluster-identifier <value> --encrypted --kms-key-id <value>
```

To remove encryption from your cluster, run the following CLI command\.

```
aws redshift modify-cluster --cluster-identifier <value> --no-encrypted
```

## Migrating to an HSM\-encrypted cluster<a name="migrating-to-an-encrypted-cluster"></a>

To migrate an unencrypted cluster to a cluster encrypted using a hardware security module \(HSM\), you create a new encrypted cluster and move your data to the new cluster\. You can't migrate to an HSM\-encrypted cluster by modifying the cluster\.

To migrate from an unencrypted cluster to an HSM\-encrypted cluster, you first unload your data from the existing, source cluster\. Then you reload the data in a new, target cluster with the chosen encryption setting\. For more information about launching an encrypted cluster, see [Amazon Redshift database encryption](working-with-db-encryption.md)\. 

During the migration process, your source cluster is available for read\-only queries until the last step\. The last step is to rename the target and source clusters, which switches endpoints so all traffic is routed to the new, target cluster\. The target cluster is unavailable until you reboot following the rename\. Suspend all data loads and other write operations on the source cluster while data is being transferred\. <a name="prepare-for-migration"></a>

**To prepare for migration**

1. Identify all the dependent systems that interact with Amazon Redshift, for example business intelligence \(BI\) tools and extract, transform, and load \(ETL\) systems\.

1. Identify validation queries to test the migration\. 

   For example, you can use the following query to find the number of user\-defined tables\.

   ```
   select count(*)
   from pg_table_def
   where schemaname != 'pg_catalog';
   ```

   The following query returns a list of all user\-defined tables and the number of rows in each table\.

   ```
   select "table", tbl_rows
   from svv_table_info;
   ```

1. Choose a good time for your migration\. To find a time when cluster usage is lowest, monitor cluster metrics such as CPU utilization and number of database connections\. For more information, see [Viewing cluster performance data](performance-metrics-perf.md)\.

1. Drop unused tables\. 

   To create a list of tables and the number of the times each table has been queried, run the following query\. 

   ```
   select database,
   schema,
   table_id,
   "table",
   round(size::float/(1024*1024)::float,2) as size,
   sortkey1,
   nvl(s.num_qs,0) num_qs
   from svv_table_info t
   left join (select tbl,
   perm_table_name,
   count(distinct query) num_qs
   from stl_scan s
   where s.userid > 1
   and   s.perm_table_name not in ('Internal worktable','S3')
   group by tbl,
   perm_table_name) s on s.tbl = t.table_id
   where t."schema" not in ('pg_internal');
   ```

1. Launch a new, encrypted cluster\. 

   Use the same port number for the target cluster as for the source cluster\. For more information about launching an encrypted cluster, see [Amazon Redshift database encryption](working-with-db-encryption.md)\. 

1. Set up the unload and load process\. 

   You can use the [Amazon Redshift Unload/Copy Utility](https://github.com/awslabs/amazon-redshift-utils/tree/master/src/UnloadCopyUtility) to help you to migrate data between clusters\. The utility exports data from the source cluster to a location on Amazon S3\. The data is encrypted with AWS KMS\. The utility then automatically imports the data into the target\. Optionally, you can use the utility to clean up Amazon S3 after migration is complete\. 

1. Run a test to verify your process and estimate how long write operations must be suspended\. 

   During the unload and load operations, maintain data consistency by suspending data loads and other write operations\. Using one of your largest tables, run through the unload and load process to help you estimate timing\. 

1. Create database objects, such as schemas, views, and tables\. To help you generate the necessary data definition language \(DDL\) statements, you can use the scripts in [AdminViews](https://github.com/awslabs/amazon-redshift-utils/tree/master/src/AdminViews) in the AWS GitHub repository\.<a name="migration-your-cluster"></a>

**To migrate your cluster**

1. Stop all ETL processes on the source cluster\. 

   To confirm that there are no write operations in process, use the Amazon Redshift Management Console to monitor write IOPS\. For more information, see [Viewing cluster performance data](performance-metrics-perf.md)\. 

1. Run the validation queries you identified earlier to collect information about the unencrypted source cluster before migration\.

1. \(Optional\) Create one workload management \(WLM\) queue to use the maximum available resources in both the source and target cluster\. For example, create a queue named `data_migrate` and configure the queue with memory of 95 percent and concurrency of 4\. For more information, see [Routing Queries to Queues Based on User Groups and Query Groups](https://docs.aws.amazon.com/redshift/latest/dg/tutorial-wlm-routing-queries-to-queues.html) in the *Amazon Redshift Database Developer Guide*\.

1. Using the `data_migrate` queue, run the UnloadCopyUtility\. 

   Monitor the UNLOAD and COPY process using the Amazon Redshift Console\. 

1. Run the validation queries again and verify that the results match the results from the source cluster\. 

1. Rename your source and target clusters to swap the endpoints\. To avoid disruption, perform this operation outside of business hours\.

1. Verify that you can connect to the target cluster using all of your SQL clients, such as ETL and reporting tools\.

1. Shut down the unencrypted source cluster\.