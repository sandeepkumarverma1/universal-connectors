# GreenplumDB-Guardium Logstash filter plug-in
### Meet Dynamodb
* Tested versions: 6.21.0
* Environment: On-premise
* Supported inputs: Filebeat (push)
* Supported Guardium versions:
  * Guardium Data Protection: 11.4 and above
  * Guardium Insights: 3.2
  * Guardium Insights SaaS: 1.0

This is a [Logstash](https://github.com/elastic/logstash) filter plug-in for the universal connector that is featured in IBM Security Guardium. It parses events and messages from the GreenplumDB log into a [Guardium record](https://github.com/IBM/universal-connectors/blob/main/common/src/main/java/com/ibm/guardium/universalconnector/commons/structures/Record.java) instance (which is a standard structure made out of several parts). Information is then sent over to the Guardium. Guardium records include the accessor (the person who tried to access the data), the session, data, and exceptions. If there are no errors, the data contains details about the query and the Guardium sniffer parses the Greenplum queries. As of now, the Greenplumdb plug-in only supports the Guardium Data Protection as of now.

The plug-in is free and open-source (Apache 2.0). It can be used as a starting point to develop additional filter plug-ins for Guardium universal connector.

## 1. Installing Greenplumdb on RHEL
### Procedure:

#### Step 1: Configure the Systems
##### Disable or configure SELinux.
1. As the root user, check the status of SELinux:
```
$ sestatus
```
2. If SELinux is not disabled, disable it by editing the /etc/selinux/config file. as root, change the value of the SELINUX parameter in the config file as follows:
SELINUX=disabled
3. If the System Security Services Daemon (SSSD) is installed on your systems, as root, edit /etc/sssd/sssd.conf and add this parameter:
selinux_provider=none
4. Reboot the system to apply any changes that you made and verify that SELinux is disabled.
```
$ sudo reboot
```

##### Create the gpadmin account.
1. Create the gpadmin group and user.
This example creates the gpadmin group, creates the gpadmin user as a system account with a home directory and as a member of the gpadmin group, and creates a password for the user.
```
$ groupadd gpadmin
$ useradd gpadmin -r -m -g gpadmin
$ passwd gpadmin
New password: <changeme>
Retype new password: <changeme>
```

2. Grant sudo access to the gpadmin user.
3. On Red Hat or CentOS, run visudo and uncomment the %wheel group entry.
```
$ sudo -i
$ visudo

```
%wheel        ALL=(ALL)       NOPASSWD: ALL

Make sure you uncomment the line that has the NOPASSWD keyword.

4. Add the gpadmin user to the wheel group with this command.
```
$ usermod -aG wheel gpadmin
```

#### Step 2: Install the Greenplum Database
##### Installation
1. Download and copy the Greenplum Database package to the gpadmin user's home directory on the master.
```
$ wget https://github.com/greenplum-db/gpdb/releases/download/6.21.0/open-source-greenplum-db-6.21.0-rhel8-x86_64.rpm
```
2. With sudo (or as root), install the Greenplum Database package.
```
sudo yum install open-source-greenplum-db-6.21.0-rhel8-x86_64.rpm
```
3. Change the owner and group of the installed files to gpadmin.
```
$ sudo chown -R gpadmin:gpadmin /usr/local/greenplum*
$ sudo chgrp -R gpadmin /usr/local/greenplum*
```
##### Enabling Password-less SSH
1. For enabling password-less SSH, we need to copy both keys(one key is generated through the below command using a pem file, and the other key is generated by keygen(from id_rsa.pub file) in an .ssh/authorized_keys file.
```
$ ssh-keygen -y -f private_key1.pem > public_key1.pub
```
2. In RHEL, the key generated by the above command is automatically added to the authorized keys. If it is not, then manually add the key. To check whether the rsa key file is already there or not, execute the following command.
```
$ ls -al ~/.ssh/id_*.pub
```
3. If the key is not available, then we need to create new one.
```
$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ec2-user/.ssh/id_rsa):
Created directory '/home/ec2-user/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```
* Note : Do not enter any value in the prompt because this will used for password-less ssh.

4. Check whether the newly-generated key file is available or not.
```
$ cat .ssh/id_rsa.pub 
```
5. After generating the id_rsa.pub file, you need to manually copy this key into an authorized_keys file.
```
$ nano .ssh/authorized_keys
```
6. After enabling the password-less ssh, we need to verify it by running the following command.
```
$ ssh localhost
Select yes when prompted to add a hostkey(yes/no).
```
7. Execute an exkeys command. If it fails, execute the source command before it/that precedes the exkeys command.
```
$ gpssh-exkeys -h localhost
$ source /usr/local/greenplum-db-6.21.0/greenplum_path.sh
```
#### Step 3: Configuring the Greenplum Database
1. Execute the below commands  for configuration-related changes.
```
$ mkdir greenplum-db-node
$ cd greenplum-db-node
$ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_singlenode .
```
2. Open gpinitsystem_singlenode using nano/vi editor and update the following fields.
```
$ nano gpinitsystem_singlenode 

MACHINE_LIST_FILE=./hostlist_singlenode
declare -a DATA_DIRECTORY=(/home/<user name>/greenplum-db-node/gpdata1)
MASTER_HOSTNAME=<hostname>
MASTER_DIRECTORY=/home/<user name>/greenplum-db-node/gpmaster
```
3. Put the hostname into the hostlist_singlenode file.
```
$ nano hostlist_singlenode
```
4. Create a directory in the greenplum-db-node/ directory and execute the following commands:
```
$ mkdir gpmaster
$ mkdir gpdata1
$ gpinitsystem -c gpinitsystem_singlenode
```
#### Step 4: Setup Profile Environment for Greenplum Database 
1. Open the profile file (such as .bashrc) in a text editor.
```
$ vi ~/.bashrc
```
2. Add lines to this file to source the greenplum_path.sh file and set the MASTER_DATA_DIRECTORY environment variable.
```
$ source /usr/local/greenplum-db-6.21.0/greenplum_path.sh
$ export MASTER_DATA_DIRECTORY=/home/ec2-user/greenplum-db-node/gpmaster/gpsne-1
```
3. Save and close the file.
4. After editing the profile file, source it to make the changes active.
```
$ source ~/.bashrc
```
##### Note: 
_you must execute the above command after each server reboot event._

## 2. Enabling the Audit Logs:
### Procedure 
1. Run the following command to start the database server.
```
$ gpstart
```
2. Modify the postressql.conf file to enable the audit logs, and execute the following command to open the file.

##### Edit
``` 
  sudo nano  ~/greenplum-db-node/gpmaster/gpsne-1/postgresql.conf
```
##### Uncomment this section:
```
  log_error_verbosity = default
  log_statement = 'all‘
```
* Note: After enabling audit logging, stop the database server using $ gpstop, start the database server using $ gpstart, and then verify that audit logging is now enabled.

3. Run the following command to stop the database server.
```
$ gpstop
```
* Note: Remember to stop the database server before turning the Greenplum database instance off. Failure to do so may corrupt the instance.

4. Run the below query to generate some audit logs. For example:
```
$ psql -d template1 -c "INSERT INTO products5 VALUES ('Cheese',10, 11);" -h localhost -p 5432
```
## 3. Viewing the audit logs
The audit logs can be viewed in this location:-
```
$ cat ~/greenplum-db-node/gpmaster/gpsne-1/pg_log/<file-name.csv>
```
After viewing the audit logs , user can download the logs from a remote machine to a local machine using the scp command:- 
```
scp -i <private key pair> <username>@<public IP>:<file source on EC2> <file destination on local>
```

## 4. Configuring Filebeat to push logs to Guardium

### a. Filebeat Installation

#### Procedure

To install Filebeat on your system, follow the steps in this topic: https://gryzli.info/2019/02/15/installing-and-configuring-filebeat-on-centos-rhel/.

### b. Filebeat Configuration

Filebeat must be configured to send the output to the chosen Logstash host and port. In addition, events are filtered out with the exclude_lines. To do this, modify the filebeat.yml file which you can find inside the folder where Filebeat is installed.You can learn more about Filebeat Configuration [here](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html#:~:text=include_lines%20edit%20A%20list%20of%20regular%20expressions%20to,the%20list.%20By%20default%2C%20all%20lines%20are%20exported.).

### Procedure

1. Configuring the input section:

```
• Locate "filebeat.inputs" in the filebeat.yml file, then add the following parameters.
    filebeat.inputs:
    - type: log
    enabled: true
    paths:
    - /home/ec2-user/greenplum-db-node/gpmaster/gpsne-1/pg_log/*.csv

• To process multi-line audit events, add the settings the same inputs section.

   multiline.pattern: '^[\d]{4}[-][\d]{1,2}[-][\d]{1,2}'
   multiline.negate: true
   multiline.match: after

• Add the tags to uniquely identify the GreenplumDB events from the rest.
   tags: ["GreenplumDB_On_Premise"]

```

2. Configuring the output section:
```
• Locate "output" in the filebeat.yml file, then add the following parameters.

• Disable Elasticsearch output by commenting it out.

• Enable Logstash output by uncommenting the Logstash section. 
For example:
  output.logstash:
    hosts: ["20.115.120.146:5085"]

• The hosts option specifies the Logstash server and the port (5001) where Logstash is configured to listen for incoming Beats connections.

•You can set any port number except 5044, 5141, and 5000 (as these are currently reserved in Guardium v11.3 and v11.4 ).
```
_For more information, see https://www.elastic.co/guide/en/beats/filebeat/current/logstash-output.html#logstash-output_


3. To learn how to start Filebeat, see https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html#start

## 5. Configuring the Greenplumdb filter in Guardium
The Guardium universal connector is the Guardium entry point for native audit logs. The Guardium universal connector identifies and parses the received events, and converts them to a standard Guardium format. The output of the Guardium universal connector is forwarded to the Guardium sniffer on the collector, for policy and auditing enforcements. Configure Guardium to read the native audit logs by customizing the Greenplumdb template.

### Before you begin

*  Configure the policies you require. See [policies](/../../#policies) for more information.
* You must have permission for the S-Tap Management role.The admin user includes this role by default.
* Download the [guardium_logstash-offline-plugin-greenplumdb.zip](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-onPremGreenplumdb-guardium/GreenplumdbOverFilebeatPackage/GreenPlumDB/guardium_logstash-offline-plugin-greenplumdb.zip) plug-in file.


### Procedure
1. On the collector, go to Setup > Tools and Views > Configure Universal Connector.
2. Enable the connector if it is already disabled, before uploading the UC.
3. Click Upload File and select the offline [guardium_logstash-offline-plugin-greenplumdb.zip](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-onPremGreenplumdb-guardium/GreenplumdbOverFilebeatPackage/GreenPlumDB/guardium_logstash-offline-plugin-greenplumdb.zip) plug-in. After it is uploaded, click OK.
4. Click the Plus icon to open the Connector Configuration dialog box.
5. Type a name in the Connector name field.
6. Update the input section to add the details from the [greenplumdb.conf](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-onPremGreenplumdb-guardium/greenplumdb.conf) file's input part, omitting the keyword "input{" at the beginning and its corresponding "}" at the end.
7. Update the filter section to add the details from the [greenplumdb.conf](https://github.com/IBM/universal-connectors/raw/main/filter-plugin/logstash-filter-onPremGreenplumdb-guardium/greenplumdb.conf) file's filter part, omitting the keyword "filter{" at the beginning and its corresponding "}" at the end.
8. The "type" fields should match in input and filter configuration sections. This field should be unique for every individual connector added.
9. The "tags" parameter in filter configuration should match the value of attribute tags configured in filebeat configuration for a connector.
10. Click Save. Guardium validates the new connector, and enables the universal connector if it was disabled. After it is validated, it appears in the Configure Universal Connector page.

## 6. Limitations

- The following important fields can't be mapped with Greenplumdb audit logs:
  - SourceProgram: Not Available with audit logs.
  - OsUser: Not Available with audit logs.
  - ClientHostName: Not Available with audit logs.

## 7. Configuring the Greenplum filter in Guardium Insights

To configure this plug-in for Guardium Insights, follow [this guide.](/docs/Guardium%20Insights/3.2.x/UC_Configuration_GI.md)

In the input configuration section, refer to the Filebeat section.
