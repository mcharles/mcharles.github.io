---
layout: post
title: "Analysis of physician referral data - [Part 1/N] Using SparkR on Amazon EC2 Cluster"
date: "2015-10-02"
---

I set out to calculate the ratio of referral counts to physicians by zip code in the United States. note that this exercise was undertaken to gain experience with an AWS Spark cluster and w/ visualization.

---
#### Data:

Referrals: [http://downloads.cms.gov/foia/physician-referrals-2012-2013-days30.zip][referrals]
</br>([data explanation][ref_expl])

NPI data: [http://download.cms.gov/nppes/NPI_Files.html][NPI] </br> ([more info][NPI_info])


In short, one dataset contains information on the number of referrals made by a physician (identified by NPI - National Provider Identifier). The other dataset provides information on an NPI, including the postal code of the NPI's practice address.

---

The unzipped .csv files used here are ~3-6GB. Using a distributed computing framework like Spark isn't strictly necessary on this scale - multi-processing can be employed on a local machine to accomplish the same task. If you have used data manipulation packages like 'dplyr' in R, the code employed here should seem familiar.

###### Note:
1. My local machine is running Ubuntu 14.04

2. The steps here will involve spending money on AWS. If you plan to use a Spark cluster on AWS, I suggest first refining your code locally (for example, read in only a few thousand rows of the referrals and NPI data into R on your local machine) to minimize the amount of up-time for the AWS instances. I spent an afternoon experimenting with this AWS environment (including destroying and re-starting the cluster several times) and the charged was approximately $5 (Oct 2015).



</br>

##### Configure AWS account
Sign-up for AWS and generate [AWS access key][AWS]. Set the key & ID as an environment variable:


{% highlight bash %}
export AWS_ACCESS_KEY_ID= <id>
export AWS_SECRET_ACCESS_KEY= <key>
{% endhighlight %}


Additionally, create a [key-pair][ec2_kp] to enable secure access to Spark cluster. Make sure the right mod permissions are assigned to the .pem file: `chmod 400 mykey.pem`


##### Download & Install Spark

There is a lot of documentation for installing Spark. The [official docs][spark_docs] are pretty comprehensive. For this exercise, I am mainly interested in the EC2 scripts (which will spin up a cluster that already has Spark installed).

##### Upload data to S3 Bucket

I used the AWS console to upload my .csv files to an S3 bucket. This can also be accomplished using [AWS CLI][aws_cli].


##### Spin-up Spark Cluster

{% highlight bash %}
$SPARK_HOME/ec2/spark-ec2 --key-pair=<key-pair-name> -i <path-to-key-pair> --region=us-east-1 --zone=us-east-1a --instance-type=m3.xlarge   --copy-aws-credentials  --slaves=2 launch spark-cluster
{% endhighlight %}

To suit the task at hand, you can specify EC2 instance type, number of Spark workers above, EBS size, etc. ([more info][ec2_docs])

Including `--copy-aws-credentials` helps when calling an S3 bucket within the same AWS account.

This process will take several minutes. You might see an error about being unable to SSH into an instance, but this is likely temporary. Let the process continue to run and the output will eventually include a link along the lines of "http://<spark-master-aws-dns>:8080". This is a web UI for the cluster. The address of the actual spark master will be listed in bold at the table of the page.

##### Copy data from S3 Bucket to ephemeral HDFS

SSH into the spark master with your .pem file, e.g.:
{% highlight bash %}
ssh -i ./aws_spark.pem ec2-user@54.173.179.133
{% endhighlight %}


The spark-ec2 script automatically configures an ephemeral HDFS instance - data in HDFS will be lost when cluster is shut down! (There is also a persistent-hdfs that can be used in place of ephemeral).

Become the root user and start all the HDFS services (a few are already turned on at boot). Then copy data from S3 to HDFS

{% highlight bash %}

sudo su
cd /root/ephemeral-hdfs
bin/start-all.sh
bin/hadoop distcp s3n://<s3_bucket_name>//<data_file> hdfs:///<destination>

{% endhighlight %}

##### Run S





[referrals]: http://downloads.cms.gov/foia/physician-referrals-2012-2013-days30.zip
[ref_expl]:  http://downloads.cms.gov/FOIA/ReferralPatterns2012-2013.pdf
[NPI]: http://download.cms.gov/nppes/NPI_Files.html
[NPI_info]: https://www.cms.gov/Regulations-and-Guidance/HIPAA-Administrative-Simplification/NationalProvIdentStand/DataDissemination.html
[AWS]: http://docs.aws.amazon.com/general/latest/gr/managing-aws-access-keys.html
[spark_docs]: http://spark.apache.org/docs/latest/
[aws_cli]: https://aws.amazon.com/cli/
[ec2_kp]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
[ec2_docs]: http://spark.apache.org/docs/latest/ec2-scripts.html
