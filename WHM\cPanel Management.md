# WHM Common Scripts to use at terminal   <br/>

__To kill dns zones__
```
/scripts/killdns example.com
```
<br/>
# How to add rds to co-location server   <br/>
From WHMCS go to client area <br/>
Select product and click manage <br/>
Find *RDNS* on the page. <br/>
Click on + icon <br/>
Add rdns to the specific IP and click save. <br/>
<br/>
# How to check VPS storage usage in Depath   <br/>
<br/>
du -h --max-depth=1
<br/>

# PHP Selector not available contact your Hoster

Visit terminal then <br/>
```licD_cloudlinux``` <br/>
If failed <br/>
Run ```licD_cloudlinux --mode``` <br/>
Select `Bespoke mode` <br/>
If license not activated then run `licD_cloudlinux` again <br/>


# 404 error because of lswscache and iptables <br/>
<br/>
<br/>

* `/usr/sbin/iptables -F`: <br/>
This command flushes (deletes) all the rules in the default filter table of iptables. It removes all the configured firewall rules for incoming and outgoing network traffic.<br/>

* `/usr/sbin/iptables -X`: <br/>
This command deletes all the user-defined chains in the default filter table. It removes any additional custom chains that were created.<br/>

* `/usr/sbin/iptables -t nat -F`: <br/>
This command flushes the rules in the NAT table of iptables. It removes all the configured network address translation rules for network packet manipulation.<br/>

* `/usr/sbin/iptables -t nat -X`: <br/>
This command deletes the user-defined chains in the NAT table. It removes any additional custom chains created in the NAT table.<br/>

* `/usr/sbin/iptables -t mangle -F`:<br/>
 This command flushes the rules in the Mangle table. The Mangle table is used for specialized packet alteration, and this command removes all the configured rules in that table.<br/>

* `/usr/sbin/iptables -t mangle -X`: <br/>
This command deletes the user-defined chains in the Mangle table. It removes any additional custom chains created in the Mangle table.
* `/usr/sbin/iptables -P INPUT ACCEPT`: This command sets the default policy for the INPUT chain to ACCEPT. It means that if no specific rule matches incoming packets, they will be accepted.<br/>

* `/usr/sbin/iptables -P FORWARD ACCEPT`: <br/>
This command sets the default policy for the FORWARD chain to ACCEPT. It means that if no specific rule matches packets being forwarded through the system, they will be accepted.
* `/usr/sbin/iptables -P OUTPUT ACCEPT`: <br/>
This command sets the default policy for the OUTPUT chain to ACCEPT. It means that if no specific rule matches outgoing packets, they will be accepted.<br/>

**Overall Effect:** <br/>


* These commands effectively reset the firewall configuration. Here's a summary of the actions:<br/>

* All existing rules from the filter, nat, and mangle tables are flushed.<br/>

* All custom chains from those tables are deleted.<br/>

* The default policy for all three chains (INPUT, FORWARD, OUTPUT) is set to allow all traffic.<br/>

<br/>

**_Note: While these commands allow all traffic, it's generally considered insecure to leave your firewall wide open. You should typically define more specific rules to control what traffic is allowed in and out of your system. This might involve creating rules that allow specific ports or protocols for desired applications and services_**

**_Additional Notes:_**
<br/>
**_The iptables command is a powerful tool that can be used to configure complex firewall rules.
It is important to thoroughly understand the iptables syntax before using it to modify your firewall configuration.
You can find more information about iptables in the official documentation: https://linux.die.net/man/8/iptables_**
<br/>

rm -f /etc/pki/ca-trust/source/anchors/litespeedtech.pem <br/>
rm -f /etc/pki/ca-trust/source/anchors/wildlitespeedtech.pem <br/>
rm -f /etc/pki/ca-trust/source/anchors/www.litespeedtech.pem <br/>
<br/>
/usr/bin/update-ca-trust
