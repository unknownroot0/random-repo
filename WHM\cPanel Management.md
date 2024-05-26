# WHM Common Scripts to use at terminal   <br/>

__ To kill dns zones __
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

# How to check VPS storage usage in Depath   <br/>

du -h --max-depth=1
<br/>

# PHP Selector not available contact your Hoster

Visit terminal then <br/>
```licD_cloudlinux``` <br/>
If failed <br/>
Run ```licD_cloudlinux --mode``` <br/>
Select `Bespoke mode` <br/>
If license not activated then run `licD_cloudlinux` again <br/>
