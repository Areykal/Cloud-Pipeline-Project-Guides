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
- 