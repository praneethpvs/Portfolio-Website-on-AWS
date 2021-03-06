# Portfolio-Website-on-AWS

The first step that i had done was to design a basic architecture that is highly available. I had limited the scope of the architecture to the free tier. The architecture that i have designed is shown below.  
  
![my website architecture](https://user-images.githubusercontent.com/31011479/29950962-4a2de93a-8e74-11e7-82dd-ad43cc1a98d5.jpg)

You can find my website here: http://pvspraneeth.com/

## Step-By-Step procedure

### STEP-1.
* The first thing that i had done was to get a domain from AWS. Since it is a registrar we can get domains directly from AWS.  

### STEP-2.
* The second step is creating a role where my Ec2 Instances will be able to communicate with the S3 buckets. This helps in improving the security of my AWS instances as my secret access key and password are not stored in the Ec2 instance.  
* _I have named the role as_ ``` ec2-s3-adminAccess ```

### STEP-3.
* Next step is to create the requires security groups for the Ec2 instance and the RDS Instance.  
* _NOTE: The RDS instance security group should accept traffic only from the Ec2 instance security group so that it is not open directly to the outside world._

**Web Security group.**  

Type | protocol | Port Range | Source
------------ | ------------- | ------------- | -------------
HTTP(80) | TCP(6) | 80 | 0.0.0.0/0
SSH(22) | TCP(6) | 22 | My IP

**Rds Security group.**  

Type | protocol | Port Range | Source
------------ | ------------- | ------------- | -------------
MySQL/Aurora(3306) | TCP(6) | 3306 | Web Security group ID

### STEP-4.
Next Setting up and Application Load Balancer. We can use either an Elastic load balancer or Applicaion load balancer. Since Applicaton Load balancer(ALB) is cheaper in price(if it exceeds free tier) i had chosen to go with ALB. Health check is done on the file ```healthy.html```.  
  
* Go back to *Route53* and create a RecordSet of your domain which points to your ***Application Load Balancer***.  
* I have setup the RecordSet name is as a Naked Domain and used an **Alias Record**_(Yes)_.
 
### STEP-5.
Now setting up the s3 buckets. One bucket is to serve the web content and the other bucket is to serve the media content using CloudFront. This helps in serving the media in an Accelerated way to the end user.  
  
* The buckets which i used are ```pvspraneeth-web-code``` and ```pvspraneeth-web-media```.  
  
### STEP-6.
Now provision a CloudFront CDN network. Since we are serving web content i had chosen to setup a ```web distribution```.
* The origin is going to be the bucket ```pvspraneeth-web-media```.  
* Enable **Restrict Bucket Access**_(yes)_ so that the users get the data from the S3 bucket that we had setit up.
* Next **Grant Read permissions on bucket**_(Yes, Update bucket Policy)_. This helps to access the media which we upload publicly.

### STEP-7.
The nect step is to provision the RDS Instance.
* I had chosen to select the **MySQL Community edition** and made this as an **Dev/Test** Instance _(Stays in free tier)_.
* Select the DB Instance class as: **db.t2.micro**
* Disable the Multi-AZ Deployment to stay in free tier.
* Choose your preferred **Db Instance Identifier, Master Username & Master Password**.
* use the **Rds Security group** which we had created earlier & Make the publicly accessible option to **No**.
* Choose you Database name and leave everything as default & Launch the DB Instance.

### STEP-8.
The next step is setting up the Ec2 Instance.
* Use the default VPC and the role which we created before``` ec2-s3-adminAccess ```.
* In the Advanced details use the ```WebserverbashScript.sh``` to setup the Ec2 Instances as a Web Server for hosting your website.
* Once your Ec2 Instance is up and running, go to the **Target groups** of your ALB and add your Ec2 Instance to the **Registered Instances**_(In target section)_. 
* Once the Instance passes the health check, you should be able to type in your domain name and the Wordpress screen shows up. Follow the Instructions on the screen and finish the word press setup.  
*Note: For the Db endpoint go the RDS Instance and copy its Endpoint*.

### STEP-9.
Now we need to enable URL Re-writes so that the content can be served from the cloud front distribution.
* Use the ```htaccess.txt``` file from the above to enable the URL Re-writes.
* Execute the following commands to enable the URL Re-write.
```
wget https://github.com/praneethpvs/Personal-Website-on-AWS/blob/master/htaccess.txt
mv htaccess.txt .htaccess
nano .htaccess
-------------------> change the cloudfront domain name and save the file.
service httpd restart
```

### STEP-10.
Setting up Automatic synchronization of your web changes to you corresponding s3 buckets.
```
cd /etc
nano crontab
```
* add the following two lines at the end of the file
```
*/5 * * * * root aws s3 sync --delete /var/www/html/wp-content/uploads s3://<Your-Media-Bucket-Name>
*/5 * * * * root aws s3 sync --delete /var/www/html s3://<Your-Code-Bucket-Name>
```
* save the file.
* we can restart the service if we like: ``` service crond restart```.

### STEP-11.
Finally placing your Ec2 Instance behind an Auto Scaling group.
* Stop your Instance and take a **snapshot** of the Ec2 Instance.
* Create a Launch configuration using the snapshot taken above.
* Now create a Auto Scaling Group from the above Launch configuration, upload the above file **{AsgUserdata.sh}** in the Advanced settings.
* place the above autoscaling group in the default vpc.
* _Enable recieve traffic from one or more Load Balancers_ and choose the target group of the original web server.
* Health checks type is done based on the ELB.
* Configure Scaling policy based on the requirements. I have chosen to keep it the default size so that it will not exceed 750 hrs of free usage.
* If required setup Notifications on your ASG as well.
