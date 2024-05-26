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
<br/>
/usr/sbin/iptables -F <br/>
/usr/sbin/iptables -X <br/>
/usr/sbin/iptables -t nat -F <br/>
/usr/sbin/iptables -t nat -X <br/>
/usr/sbin/iptables -t mangle -F <br/>
/usr/sbin/iptables -t mangle -X <br/>
/usr/sbin/iptables -P INPUT ACCEPT <br/>
/usr/sbin/iptables -P FORWARD ACCEPT <br/>
/usr/sbin/iptables -P OUTPUT ACCEPT <br/>
<br/>
rm -f /etc/pki/ca-trust/source/anchors/litespeedtech.pem <br/>
rm -f /etc/pki/ca-trust/source/anchors/wildlitespeedtech.pem <br/>
rm -f /etc/pki/ca-trust/source/anchors/www.litespeedtech.pem <br/>
<br/>
/usr/bin/update-ca-trust
