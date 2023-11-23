- **Phase 1: Design the architectural diagram of your solution and estimate its cost.**
- ### Phase 2:  
	- **Task 1: Familiarize yourself with the set up and environment of the project.**
	- Go through the different services to verify there exists:
		- A virtual private cloud (VPC) named _Lab VPC_ with a public subnet named _Public Subnet_ in it. The subnet can reach the internet through an internet gateway.
		- An AWS Cloud9 instance that runs as an EC2 instance in the public subnet. AWS Cloud9 provides an integrated development environment (IDE), which is convenient to edit files and run AWS Command Line Interface (AWS CLI) commands
		- A security group that is associated with the EC2 instance.
		- An IAM role named _CafeRole_, which is attached to the EC2 instance. The role allows access from the instance to other AWS services, such as AWS Systems Manager, Amazon S3, and CloudWatch.
		- The AWS Cloud9 instance also hosts the café web application. The web application runs on an Apache httpd web server and PHP, and it uses a MariaDB database that is also running on the instance.
	- **Task 2: Modify a security group and verifying that the web app loads:**
		1. In the service menu, search **EC2**.
		2. On the side menu, scroll down to **Security Groups**.
		3. Select **aws-cloud9-Cloud9-Lab-IDE**.
		4. On the bottom portion of the screen, go to **Inbound rules** tab.
		5. Click **Edit inbound rules.**
		6. **DO NOT** delete the existing rules. Click **Add rule**.
		7. Type: `HTTP`
		8. Source type: `Anywhere-IPv4`
		9. On the side menu, go back to **Instances**.
		10. Select **aws-cloud9-Cloud9-Lab-IDE**.
		11. On the bottom portion of the screen, still in **Details** tab, copy the _Public IPv4 address_.
		12. Paste the IP address into a new browser tab and add `/cafe` to the end of the URL. Make sure you are loading `http` and not `https`. Keep the browser tab open.
	- **Task 3: Observing and backing up the httpd `access_log`**
		1. In the service menu, search **Cloud9**.
		2. In the **Environments** page, under **Cloud9 IDE** click **Open**.
		3. Track changes in real time to the Apache web server's `access_log` file: 
			`sudo tail -f /var/log/httpd/access_log`
		4. Return to the browser tab where the cafe web app is open and refresh the page.
		5. Confirm that each time you load a page on the website, a new log entry is created in the `access_log`.
		6. To stop the `tail` command, press `Ctrl+C`.
		7. To back up the `access_log` file, run the following command:
			`sudo cp /var/log/httpd/access_log /home/ec2 user/environment/initial_access_log`
		 
- **Phase 3: Installing the CloudWatch Agent and creating the config file**:
	- **Task 1: Installing the CloudWatch Agent on the web server**
		`sudo yum install -y amazon-cloudwatch-agent`
	- **Task 2: Creating the config file for the CloudWatch agent**
		1. Download and install a CloudWatch agent configuration file.
```bash
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP4-1-79925/capstone-4-clickstream/s3/config.json
sudo mv config.json /opt/aws/amazon-cloudwatch-agent/bin/
```

2. Review the contents of the `config.json` file.
`sudo cat /opt/aws/amazon-cloudwatch-agent/bin/config.json`

- **Task 3: Configuring the `httpd.conf` log format as JSON**
	1. Locate the web server's config file, and back up the file.
		`sudo cp /etc/httpd/conf/httpd.conf <destination directory>`
	**NOTE**: replace destination directory with any directory within environment. EX:  `sudo cp /etc/httpd/conf/httpd.conf ./testDir`
	2. modify the `httpd.conf` file so that the error logs and access logs will be formatted as JSON instead of the default CLF format
		- To avoid the need to use a terminal text editor (such as vi or nano) to edit the files in the _/etc/httpd/conf_ directory, run the following commands:
			`ln -s /etc/httpd/conf /home/ec2-user/environment/httpdconf`
		**Note:** The previous command creates a symlink to the _/etc/httpd/conf_ directory so that you can see the files in the AWS Cloud9 **Environment** window.
			`sudo chown -R ec2-user /etc/httpd/conf`
		- In the **Environment** window of the IDE, expand the **httpdconf** folder, and open the **httpd.conf** file in the file editor.
	3. Adjust the error log configuration:
	    - In the _httpd.conf_ file, go to the line that reads `ErrorLog "logs/error_log"`(around line 182). Comment out the line.
	        **Tip:** To comment out a line, enter a `#` character at the beginning of the line.
	    - Copy and paste the following text as a new line immediately after the line that you just commented out:
	      `  ErrorLog "/var/log/www/error/error_log"`
	    - Copy and paste the following text as a new line immediately after the `ErrorLog` line that you just added:
	        `ErrorLogFormat "{\"time\":\"%{%usec_frac}t\", \"function\" : \"[%-m:%l]\", \"process\" : \"[pid%P]\" ,\"message\" : \"%M\"}"`
	4. Adjust the access log configuration:
	    - Go to the line that reads `<IfModule log_config_module>` (around line 191). Comment out _**the line below it**_, which contains `LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined`.
	    - Copy and paste the following text as a new line immediately after the `LogFormat "%h %l %u %t \"%r\" %>s %b" common` line that remains:    
		 `LogFormat "{ \"time\":\"%{%Y-%m-%d}tT%{%T}t.%{msec_frac}tZ\", \"process\":\"%D\", \"filename\":\"%f\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\",\"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\",\"referer\":\"%{Referer}i\"}" cloudwatch`
	    - Comment out the entire `<IfModule logio_module>` section. To do this, comment out the three lines that weren't previously commented out.
	    - Go to the line that reads `CustomLog "logs/access_log" combined` (around line 219).
	    - Keep the line, but copy and paste the following text as a new line immediately after the `CustomLog` line.
	        `CustomLog "/var/log/www/access/access_log" cloudwatch`
	5. Save and close the file.
- **Task 4: Using the updated configuration file for the CloudWatch agent**
	1. Create new access and error log directories
		- Navigate to /var/log directory.
		- Create `www` directory: `mkdir www`
		- Navigate to `www`: `cd www`
		- Create `access` directory: `mkdir access`
		- Create `error` directory: `mkdir error`
	2. Restart the httpd service:
		`sudo systemctl restart httpd`
	3. Start the CloudWatch Agent so that it uses the new `config.json` file
		`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`
	4. To confirm that the agent is now running and active, run the following command: 
	    `service amazon-cloudwatch-agent statu
	    
	    s` 
- **Phase 4: Testing the CloudWatch agent**
	- **Task 1:  Observing the httdp access_log file**
		1. In the café web application, perform a variety of actions, as you did before to generate logs.
		2. Use the same technique that you used in phase 2 task 3 to observe the actions as they are added to the access_log file. Remember that you configured the web server to write to a different log location, and log entries should now be written in JSON format instead of the default Common Log Format (CLF).
	- **Task 2: Observing the amazon-cloudwatch-agent.log file**
		1. Determine the location of the _amazon-cloudwatch-agent.log_ file, and use the `cat` command to see the contents of the file.
			`sudo cat /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
		2. In the file, look for lines that are similar to the following:
```bash
	[inputs.logfile] Reading from offset 10620 in /var/log/www/error/error_log
    [inputs.logfile] Reading from offset 19850 in /var/log/www/access/access_log
    [logagent] piping log from apache/error/i-0d378cc2f1bb61140(/var/log/www/error/error_log) to cloudwatchlogs with retention 180
    [logagent] piping log from apache/access/i-0d378cc2f1bb61140(/var/log/www/access/access_log) to cloudwatchlogs with retention 180
```
These lines indicate that the CloudWatch agent is reading from the two web server log files and piping (sending) those logs to the CloudWatch service. (Ort kernh kor ort ey dae lol).
- **Task 3: Placing an order in the web application and observing the CloudWatch logs**
	1. Open the café web application at `http://<public-ip-address>/cafe`
	    **Note:** Replace <_public-ip-address_> with the public IP address of the web server instance.
	    **Tip:** To make the log entries a bit different, use a mobile device to browse the website.
	2. Go to the **Menu** page, and submit an order.
	3. Verify that your latest actions were captured in CloudWatch:
	    - In the CloudWatch console, locate the **apache/access** entry in the **Log groups** area.
	    - Open the log stream.
	    - Expand the first entry to see the log details, which are in JSON format. Verify that your latest actions were captured.
- **Phase 5: Using the simulated log and ensuring that CloudWatch receives the entries**
	- **Task1: Analyzing the simulated log file**
		1. Review the simulated log file:
		    - To see the first few lines of the simulated log data, run the following command:
		       `cat samplelogs/access_log.log | head`
		    - To pretty-print the first line of the log file so that it's easier to read, run the following command:
		      `cat samplelogs/access_log.log | head -1 | python -m json.tool`
		        Notice that the format of the file matches the format of the logs that your web server generated after you changed the LogFormat setting in the httpd.conf file. This format matching is intentional.
		    - To count the number of lines in the file, run the following command:
		      `cat samplelogs/access_log.log| wc -l`
		        As you can see, this log has a lot of data in it.
	- **Task 2: Using the new log file**
		1. Stop the CloudWatch agent service: 
			`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a stop`
		2. Put the new simulated access log in the location where the CloudWatch agent expects to find it:
			- Verify that you are in `/environment` directory.
			- Rename and move the simulated `access_log` file to the location where CloudWatch agent expects to find it.
			`sudo mv ./samplelogs/access_log.log /var/log/www/access/access_log`
			- Restart the CloudWatch Agent service so that it will send the new log entries to CloudWatch. 
			`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`
	- **Task 3: Confirming that the new logs appear in the CloudWatch log group**
		Now that the new log file is in place and the CloudWatch agent service has been restarted, you will verify that the logs appear in CloudWatch.
		
		In the CloudWatch console, locate the **apache/access** log group. In the log stream, confirm that many more entries appear in the stream than you saw previously.
		
		**Tip:** An indication that the log group now contains the new log data is if the stream has multiple entries with the same timestamp.
		
		You have confirmed that the simulated log data is now appearing in the log stream. 
- **Phase 6: Using CloudWatch Logs Insights for analysis**
	- **Task 1: Determining the number of visitors who accessed the menu**
		1. In the navigation pane of the CloudWatch console, choose **Logs Insights**.
		    **Tip:** For this task and later tasks, you might want to refer to [Analyzing Log Data with CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html).
		2. Get the count of users who accessed the menu:
		    - Set the time period of the query to match the time period that the access_log has data for.
		    - Run the following query:
		        
			fields @timestamp, remoteIP
			| filter request = "/cafe/menu.php"
			| count(remoteIP) as Visitors by @timestamp
			| sort @timestamp asc`
		        
			The results are summarized on the **Logs** tab with a message in the following format (where X and Y are number values):
		        Showing 1000 of X records matched
		        Y records...scanned...
			The key data to notice is the number of records that matched (X) out of the total number of records that were scanned (Y).
		    - Record this information in a new text file in the AWS Cloud9 IDE. Name the file `phase6-results.txt` and save it to the `~/environment` directory. Alternatively, if your instructor has asked you to create a presentation or document to record your work, include the information there.
		3. Save the query with the following settings:
		    - **Query name:** Enter `menu-visitors`
		    - **Folder:** Choose **Create new** and then enter `non-geo-results`
		        Notice that the saved query appears in the **Queries** panel.
	- **Task 2: Determining the number of visitors who made a purchase**

		1. Get the count of users who made a purchase:
		    
		    - Set the time period of the query to match the time period that the access_log has data for.
		        
		    - Run query from the previous task, but change the filter request to `/cafe/processOrder.php`
		        
		    - Record the results in your _phase6-results.txt_ file, presentation, or document.
		        
		2. Save the query to the _non-geo-results_ folder. Use `purchasers` for the query name.
	- **Task 3: Determining the number of visitors who accessed the menu but didn't make a purchase**

		1. Use the results of the prior two tasks to determine how many people accessed the menu page but didn't make a purchase.
		    
		    **Tip:** The formula to get this count is as follows: _Number of visitors who accessed the menu but didn't make a purchase = (Number of visitors who accessed the menu) - (Number of visitors who made a purchase)_
		    
		2. Calculate the percentage of people who visited the menu page and also made a purchase.
		    
		3. Record these results in your _phase6-results.txt_ file, presentation, or document.
		    
		You have successfully used queries in CloudWatch Logs Insights to determine how many website visitors completed certain actions.
- **Phase 7: Adjusting the pipeline to deliver new insights**
	In this final phase of the project, you will enhance the data in your pipeline to deliver new insights to the café owner.
	- **Task** 1: Understanding the requirements
		You meet with the owner of the café and share the insights that you have gathered from the clickstream data. The owner is impressed with the solution that you have built so far, but they want to know more.
		The café owner asks if you can modify the solution to provide the following:
		- Geolocation information for website visitors.
		- A visual dashboard to gain insights. The dashboard should include the following information:
		    - A _pie chart_ that shows the 10 cities that had the most website visitors who accessed the menu page.
		    - A _log table_ that shows the 10 cities that had the most website visitors who made a purchase.
		    - A _pie chart_ that shows the 10 regions that had the most website visitors who accessed the main page of the website.
		    - A _bar chart_ that shows the 10 regions that had the most website visitors who made a purchase.
		- The ability to store the access logs in Amazon S3 and the ability to query the logs by using SQL
		The geolocation information and visual dashboard will help the owner to analyze business trends and determine where to open new café locations. Being able to store the data in Amazon S3 and use SQL to query the data will provide flexibility to use a variety of tools and services to analyze the data in the future.
	
		You have new solution requirements. The remaining tasks in this phase provide additional detail to help you fulfill these requirements.
	- **Task 2: Using the example logs that include geolocation information**
		In this task, you will replace the existing access_log with a file that includes geolocation information.
		1. To analyze the sample access_log that includes geolocation information, run the following commands. The results will provide information about the data structure and the number of lines in the file:
		
		    `cd ~/environment`
		    `head -1 samplelogs/access_log_geo.log | python -m json.tool`
		    `cat samplelogs/access_log_geo.log | wc -l`
		    
		2. Replace the existing access_log on the web server with the new log that contains geolocation information. Ensure that the CloudWatch agent will capture the data and send it to CloudWatch.
	
			`sudo mv ./samplelogs/access_log_geo.log /var/log/www/access/access_log`
		    
		    Recall that you performed this task in phase 5, tasks 2 and 3. The differences this time are that you are going to have geolocation information in the new access_log, and you need to delete the existing CloudWatch log stream before you restart the CloudWatch agent.
		    
		    At a high level, the steps are as follows:
		    - Stop the CloudWatch agent.
		    - Copy the new log to the location where the CloudWatch agent expects to find it. Ensure that the new log is named `access_log`.
		    - To confirm that the access_log file was replaced, use the following command:
		        
		        `sudo head -1 /var/log/www/access/access_log`
		        
		    - Verify that you see geolocation data ("lat" and "lon") in the log entries. The following is an example entry:
		        
		       ` { "time":"2023-02-22 03:38:18", "process":"5112", "filename":"/var/www/html/cafe", "remoteIP":"91.192.109.163", "host":"10.0.1.102", "request":"/cafe", "query":"", "method":"GET", "status":"200", "useragent":"Mozilla/5.0 (Windows NT 10.0) AppleWebKit/531.2 (KHTML, like Gecko) Chrome/111.0.553.2.0.854.0", "referer":"-", "country":"ES", "region":"Andalusia", "city":"La Rinconada", "lat": "37.4867198", "lon":"-5.9814868"}`
		        
		3. Delete the existing apache/access log group from CloudWatch logs, recreate the apache/access log group and restart the CloudWatch agent service. 
	- **Task 3: Building a dashboard to observe the geolocation data in CloudWatch Logs Insights**
		1. Create a CloudWatch dashboard that is named `cafe-dashboard`
		2. **Add a pie chart widget to the dashboard** to show which cities have the most visitors to the café website's menu page. Use the following settings:
			- Widget type: **Pie**
			- Data source: **Logs**
			- Time period to query: **Custom** (Date that you started the project to today)
			- Log groups to query: **apache/access**
			- Query:
```bash
fields remoteIP, city
| filter request = "/cafe/menu.php"
| stats count() as menupopular by city 
| sort menupopular desc
| limit 10
```
- Run the query. The pie chart appears.
- Choose **Create widget**. The widget is added to the dashboard.
- Hover on the **Log group: apache/access** title of the widget and choose the edit icon.
- Rename the widget as `Cities visiting the menu the most`
- Choose **Apply**.
- On the dashboard page, choose **Save** in the top-right corner.
	3. Run the query. The pie chart appears.
	- Choose **Create widget**. The widget is added to the dashboard.
	- Hover on the **Log group: apache/access** title of the widget and choose the edit icon.
	- Rename the widget as `Cities visiting the menu the most`
	- Choose **Apply**.
	- On the dashboard page, choose **Save** in the top-right corner.
	4. Use the same approach to **add a third widget**. The widget should show the 10 regions that had the most website visitors who accessed the main page of the website:
		- Use a pie chart.
		- Name the widget as `Regions visiting the website the most`
		- The widget should cover the same time period as the first widget.
		- The query should filter requests to the `/cafe` page.

	5. Use the same approach to **add a fourth widget**. The widget should show the 10 regions that had the most website visitors who made a purchase:
	    - Use a bar chart.
	    - Name the widget as `Regions placing the most orders`.
	    - The widget should cover the same time period as the first widget.
	6. After you add all four widgets to the dashboard, make sure that you save the dashboard. 
	- #### Task 4: Saving the log file to an S3 bucket
		- Search for S3 in the services tab, copy the name of the bucket
		- In the Cloud9 terminal:
		`aws s3 cp /var/log/www/access/access_log s3://<yourBucketName>`
		Replace `<yourBucketName>` with the name you copied
	- #### Task 5: Using S3 Select to query logs that are stored in Amazon S3
		In this task, you complete the final requirement of the solution.
		
		Use S3 Select to query the logs that are now stored in Amazon S3. Compare the results from S3 Select to the CloudWatch Insights results.
		
		**Tip:** The [Filtering and Retrieving Data Using Amazon S3 Select](https://docs.aws.amazon.com/AmazonS3/latest/userguide/selecting-content-from-objects.html) page might be helpful to refer to.
		
		You have successfully implemented geolocation data in your clickstream solution. You built a dashboard to get insights from the new data, and you adjusted the solution to provide flexibility to use other services and features to analyze the data in the future.
		
