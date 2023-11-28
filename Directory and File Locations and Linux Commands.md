## The directory and file locations and Linux commands are stored in the same order as they appear in the instruction

### Anytime you need access to a particular `directory`:
	`sudo chown -R ec2-user <directory>`
### If any normal command doesn't work initially, try adding `sudo` at the beginning of that command.

- `access_log` location: 
	`/var/log/httpd/access_log`
	
- Track `access_log` file:
	`sudo tail -f /var/log/httpd/access_log`
	- This command doesn't stop by itself. To stop press `Ctrl+C`
	
- Back up the `access_log` file:
	- Command:
		`sudo cp /var/log/httpd/access_log /home/ec2-user/environment/initial_access_log`
		
- Install CloudWatch agent:
	`sudo yum install -y amazon-cloudwatch-agent`
	
- Creating config file:
	`wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP4-1-79925/capstone-4-clickstream/s3/config.json`
	
- Move the file to a new location:
	`sudo mv config.json /opt/aws/amazon-cloudwatch-agent/bin/`
	
- Review the contents of the `config.json` file:
	`sudo cat /opt/aws/amazon-cloudwatch-agent/bin/config.json`
	
- Apache web server's config file location:
	`/etc/httpd/conf/httpd.conf`
	
- Back up the file:
	`cd ~/environment`
	`sudo cp /etc/httpd/conf/httpd.conf ./back-up-config-file.conf`
	
- Create symbolic link to a directory:
	`ln -s /etc/httpd/conf /home/ec2-user/environment/httpdconf`
	
- Change ownership of the files:
	`sudo chown -R ec2-user /etc/httpd/conf`
	
- Make the access and error directory:
	`sudo mkdir /var/log/www/access`
	`sudo mkdir /var/log/www/error`
	
- Restart the httpd service using `systemctl`:
	`sudo systemctl restart httpd`
	
- Start the CloudWatch Agent:
	`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`
	
- Confirm that the agent is running and active:
	`service amazon-cloudwatch-agent status`
	
- `amazon-cloudwatch-agent.log` file location:
	`/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
	
- Print the contents of `amazon-cloudwatch-agent.log`:
	`sudo cat /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
	
- Review the simulated log file:
	`cat samplelogs/access_log.log | head`
	
- Pretty-print the first line of the log file:
	`cat samplelogs/access_log.log | head -1 | python -m json.tool`
	
- Count the number of lines in the file:
	`cat samplelogs/access_log.log| wc -l
	`
- Stop the CloudWatch agent service:
	`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a stop`
	
- Move the new simulated access log file to the right location:
	`cd ~/environment`
	`sudo mv ./samplelogs/access_log.log /var/log/www/access/access_log`
	
- Restart the CloudWatch agent service:
	`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`
	
- Make a new directory for the `phase6-results.txt`:
	`cd ~/environment`
	`mkdir phase6-result/phase6-results.txt`
	
- Analyze the sample `access_log_geo.log` file:
	`cd ~/environment`
	`head -1 samplelogs/access_log_geo.log | python -m json.tool`
	`cat samplelogs/access_log_geo.log | wc -l`
	
- Replace the existing `access_log` with the new `access_log_geo.log` file:
	`cd ~/environment`
	`sudo mv ./samplelogs/access_log_geo.log /var/log/www/access/access_log`
	
- Copy the access log from `/var/log/www/access` to the S3 bucket that was provided.
	- Go to S3 and copy the name of your bucket.
		`aws s3 cp /var/log/www/access/access_log s3://<yourBucketName>`
