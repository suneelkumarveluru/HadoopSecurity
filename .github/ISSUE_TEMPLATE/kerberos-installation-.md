---
name: 'kerberos Installation '
about: Suggest an idea for this project

---

1.	prerequisites

1)	Set up and configure Hadoop (HDP 2.x) with all necessary services.
2)	Make sure that all hosts in the realm time-synchronized.

2.	Enable Domain and Trust in Active Directory
1)	Configure network security and enable network encryption for Kerberos in the active or default local domain policy

2)	In the Active Directory console, go to:
Server Manager > Group Policy Management > Domain > Group Policy Objects > Default or Active Domain Policy and Edit
<image>

3)	Navigate to below mentioned path, and chose all the necessary encryptions.
Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options and configure Network security: Configure Encryption types allowed for Kerberos.
<image>

4)	Run these commands in PowerShell to configure Windows Kerberos with realms in Linux.
ksetup /addkdc NAMENODE.COM namenode.com
netdom trust NAMENODE.COM /Domain: IDA.COM /add /realm /passwordt:Siemens1234
ksetup /SetEncTypeAttr NAMENODE.COM DES-CBC-CRC DES-CBC-MD5 RC4-HMAC-MD5 AES128-CTS-HMAC-SHA1-96 AES256-CTS-HMAC-SHA1-96
(The password should be same as that of realm krbtgt/NAMENODE.COM@IDA.COM)


3.	Configure Kerberos KDC

1)	Install Kerberos with latest version on NameNode. 
yum install krb5-server krb5-libs krb5-auth-dialog krb5-workstation

2)	Install Kerberos with latest version on all other nodes in the cluster
yum -y install krb5-libs krb5-auth-dialog krb5-workstation

3)	Edit Kerberos configuration file (/etc/krb5.conf) as file attached below.
Where, HDP Cluster realm = NAMENODE.COM
And    Windows Domain = IDA.COM
 
4)	/usr/sbin/kdb5_util create -s

5)	Edit KDC configuration file (/var/Kerberos/krb5kdc/kdc.conf) as file attached below.
[kdcdefaults]
  kdc_port = 88
  kdc_tcp_ports = 88
[realms]
  HADOOP.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }

6)	Enable security from Ambari
a.	Open Ambari from http://namenode.com:8080/
b.	Username admin, Password admin.
c.	Go to Admin  Security  Enable security
(If security is enabled by default then disable and again enable it) 
d.	In Ambari Security Wizard edit your cluster realm name, in our case, it is NAMENODE.COM
 
e.	Click next to create principals and Keytabs page, which at the bottom has an option to Download CSV containing all the required information of principals and keytabs you’ll need in order to run the Kerberos script.
 
/var/lib/ambari-server/resources/scripts/kerberos-setup.sh /home/hadoop-master/Downloads/host-principal-keytab-list.csv ~/.ssh/id_rsa
f.	Note:  The kerberos-setup.sh script contains some unnecessary / repeated steps that may need to comment out as shown below: 
 
g.	Ensure that /etc/security/keytabs/contains updated keytabs files for the Hadoop services installed on that host.
h.	Note: if the Enable security fails at step “Check HBase” with error massages “Insufficient access on 'ambarismoketest' ”, then you need to drop and create the HBase table with valid access and retry running.
#sudo -u hbase hbase shell
>scan 'ambarismoketest'
>disable 'ambarismoketest'
>drop 'ambarismoketest'
>create 'ambarismoketest', {NAME => 'family'}
>put 'ambarismoketest', 'row01', 'family:col01', '<copy_this value_from_scan_output>'
>grant  'ambari-qa', 'RWCXA' , 'ambarismoketest', 'family' 

i.	Note: If at any step, it fails with massage “time out error”, then click on Retry button.

7)	 Create a Kerberos administrator
a.	Run the command below and provide a strong password.
/usr/sbin/kadmin.local -q "addprinc root/admin”
(If principle already exists delete it with delprinc option)

b.	Make sure the ACL file (/var/kerberos/krb5kdc/kadm5.acl) have correct realm name as shown below.
*/admin@NAMENODE.COM *

c.	Add the Kerberos key distribution center account called krbtgt
kadmin.local

addprinc -e "aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal" krbtgt/NAMENODE.COM@IDA.COM
d.	Obtain a Kerberos ticket with admin user.
kinit root/admin@NAMENODE.COM

8)	restart Kerberos services:
/sbin/service krb5kdc restart
/sbin/service kadmin restart

9)	Add rule with variable hadoop.security.auth_to_local by adding RULE:[1:$1@$0](.*@IDA.COM)s/@.*//

 

10)	Restart the Hadoop services from Ambari.
 


11)	 Authenticate users from the AD and get tickets. Initialize user by running the command below and assign the AD user password.
kinit SV186032@IDA.COM

12)	Check the creation of Kerberos TGT by running klist command. Output should like this:
<image>

13)	Test by accessing the Hadoop services from kerberised user. E.g.
Sudo –u SV186032 hadoop fs –ls /
