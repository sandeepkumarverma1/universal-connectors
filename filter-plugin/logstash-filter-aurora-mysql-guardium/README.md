# Aurora-MySQL-Guardium Logstash filter plug-in
### Meet Aurora-MySQL
* Tested versions: 2.07.2
* Environment: AWS
* Supported inputs: CloudWatch (pull)
* Supported Guardium versions:
  * Guardium Data Protection: 11.4 and above
  * Guardium Insights SaaS: 1.0

This is a [Logstash](https://github.com/elastic/logstash) filter plug-in for the universal connector that is featured in IBM Security Guardium. It parses events and messages from the aurora-mysql audit log into a [Guardium record](https://github.com/IBM/universal-connectors/blob/main/common/src/main/java/com/ibm/guardium/universalconnector/commons/structures/Record.java) instance (which is a standard structure made out of several parts). The information is then sent over to Guardium. Guardium records include the accessor (the person who tried to access the data), the session, data, and exceptions. If there are no errors, the data contains details about the query "construct". The construct details the main action (verb) and collections (objects) involved.

Currently, this plug-in will work only on IBM Security Guardium Data Protection, not Guardium Insights.

The plug-in is free and open-source (Apache 2.0). It can be used as a starting point to develop additional filter plug-ins for Guardium universal connector.


## 1. Configuring the AWS aurora-mysql service

### Procedure:
1. Go to https://console.aws.amazon.com/.
2. Click Services.
3. In the Database section, click RDS.
4. Select the region in the top right corner.
5. In the central panel of the Amazon RDS Dashboard, click Create database.
6. Choose a database creation method.
7. In the Engine options, select Amazon Aurora, and then select the Amazon Aurora MySQL-Compatible Edition.
8. Select an Capacity type(Provisioned).
9. Select template (Production,Dev/Test)
10. In the Settings section, type the database instance name and create the master account with the username and password to log in to the database.
11. Select the database instance size according to your requirements.
12. Select appropriate storage options (for example, you may want to enable auto scaling).
13. Select the Availability and durability options.
14. Select the connectivity settings that are appropriate for your environment. To make the database accessible, set the Public access option to Publicly Accessible within Additional Configuration.
15. Select the type of Authentication for the database (choose from Password Authentication, Password and IAM database authentication, and Password and Kerberos authentication).
16. Expand the Additional Configuration options:
		a. Configure the database options.
		b. Select DB cluster parameter group.
		b. Select options for Backup.
		c. If desired, enable Encryption on the database instances.
		d. In Log exports, select the log types to publish to Amazon CloudWatch (Audit log).
		e. Select the options for Deletion protection.
17. Click Create Database.
18. To view the database, click Databases under Amazon RDS in the left panel.
19. To authorize inbound traffic, edit the security group:
		a. In the database summary page, select the Connectivity and Security tab. Under Security, click VPC security group.
		b. Click the group name that you selected while creating database (each database has one active group).
		c. In the Inbound rule section, choose to edit the inbound rules.
		d. Set this rule:
			• Type: MYSQL/Aurora
			• Protocol: TCP
			• Port Range: 3306
			(depending on your requirements, the Source can be set to a specific IP address or it can be opened to all hosts)
		e. Click Add Rule and then click Save changes.
		The database may need to be restarted.

## 2. Enabling Auditing

1. Click on Parameter Groups.
2. Click on "Create Parameter Groups" Button .
3. Provide below details:
		• Parameter group family : Provide aurora-mysql version
		• Type : DB cluster parameter group
		• Group name : Name of Group
		• Description : Privide description
4. then ,Click on "create" Button.
5. Select "DB Parameter" ,Click on "Parameter group actions" ,Select "Edit".
6. Change the value of parameter,Add these settings:
		• server_audit_events = CONNECT,QUERY_DCL,QUERY_DDL,QUERY_DML	
		• server_audit_excl_users =	rdsadmin
		• server_audit_logging	= 1
		• server_audit_logs_upload	= 1
		• log_output = FILE
7. Click on "Save changes" Button.
8. Go To "Database Clustor" ,Click on "modify"
9. Go To "Additional Configuration" ,then "Database options".
10. Change DB clustor parameter group.
11. Click on continue and select Apply immediately
12. Click on Modify Cluster.
13. Reboot the DB Cluster for the changes to take effect.
		
## 3. Viewing the Audit logs

The Audit logs can be seen in log files in RDS, and also on CloudWatch.
	• “Viewing the auditing details in RDS log files”
	• “Viewing the logs entries on CloudWatch”

### Viewing the auditing details in RDS log files

The RDS log files can be viewed, watched, and downloaded. The name of the RDS log file is modifiable and is controlled by parameter log_filename.

#### Procedure
	1. Go to Services > Database > RDS > Databases.
	2. Select the database instance.
	3. Select the Logs & Events section.
	4. The end of the Logs section lists the files that contain the auditing details. The newest file is the last page.

### Viewing the logs entries on CloudWatch

By default, each database instance has an associated log group with a name in this format: /aws/rds/instance/<instance_name>/aurora-mysqlql. You can use this log group, or you can create a new one and associate it with the database instance.

#### Procedure
	1. On the AWS Console page, open the Services menu.
	2. Enter the CloudWatch string in the search box.
	3. Click CloudWatch to redirect to the CloudWatch dashboard.
	4. In the left panel, select Logs.
	5. Click Log Groups.
	

#### Limitations
	• The aurora-mysql plug-in does not support IPV6.
	• The aurora-mysql auditing does not audit Procedure,Function,Show tables operations.
	• Source Program will be seen as blank in report.

## 4. Configuring the aurora-mysql filters in Guardium

The Guardium universal connector is the Guardium entry point for native audit logs. The Guardium universal connector identifies and parses the received events, and converts them to a standard Guardium format. The output of the Guardium universal connector is forwarded to the Guardium sniffer on the collector, for policy and auditing enforcements. Configure Guardium to read the native audit logs by customizing the aurora-mysql template.

### Authorizing outgoing traffic from AWS to Guardium

#### Procedure
	1. Log in to the Guardium API.
	2. Issue these commands:
		• grdapi add_domain_to_universal_connector_allowed_domains domain=amazonaws.com
		• grdapi add_domain_to_universal_connector_allowed_domains domain=amazon.com

#### Before you begin
•  Configure the policies you require. See [policies](/../../#policies) for more information.

• You must have permission for the S-Tap Management role. The admin user includes this role by default.
	
• Download the [Aurora-Mysql-offlinePlugin.zip](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-aurora-mysql-guardium/AuroraMysqlOverCloudwatchPackage/AuroraMysql/Aurora-Mysql-offlinePlugin.zip) plug-in.

#### Procedure
1. On the collector, go to Setup > Tools and Views > Configure Universal Connector.
2. First enable the Universal Guardium connector, if it is disabled already.
3. Click Upload File and select the offline [Aurora-Mysql-offlinePlugin.zip](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-aurora-mysql-guardium/AuroraMysqlOverCloudwatchPackage/AuroraMysql/Aurora-Mysql-offlinePlugin.zip) plug-in. After it is uploaded, click OK.						 
4. Click the Plus sign to open the Connector Configuration dialog box.
5. Type a name in the Connector name field.
6. Update the input section to add the details from [auroraMysqlCloudwatch.conf](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-aurora-mysql-guardium/auroraMysqlCloudwatch.conf) file's input part, omitting the keyword "input{" at the beginning and its corresponding "}" at the end.
7. Update the filter section to add the details from [auroraMysqlCloudwatch.conf](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-aurora-mysql-guardium/auroraMysqlCloudwatch.conf)  file's filter part, omitting the keyword "filter{" at the beginning and its corresponding "}" at the end.
8. The "type" fields should match in the input and the filter configuration sections. This field should be unique for every individual connector added.
9. Click Save. Guardium validates the new connector, and enables the universal connector if it was
	disabled. After it is validated, it appears in the Configure Universal Connector page.

## Configuring the Aurora-MySQL Guardium Logstash filters in Guardium Insights

To configure this plug-in for Guardium Insights, follow [this guide.](/docs/Guardium%20Insights/3.2.x/UC_Configuration_GI.md)

For the input configuration step, refer to the [CloudWatch_logs section](/docs/Guardium%20Insights/3.2.x/UC_Configuration_GI.md#configuring-a-CloudWatch-input-plug-in).