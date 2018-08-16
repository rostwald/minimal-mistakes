One of the most annoying "features" added to mail clients during the last few years is the "Threaded View" of folders. In most cases this only means that a new email is viewed somewhere way down the list below another stone-old message from the same sender. I've "lost" several emails for months due to this stupid misfeature and claws-mail doesn't have any option to globally disable this crap (but enables it by default!).  

The folder settings however, are stored in ~/.claws-mail/folderlist.xml, so nuking that option for all folders can be achieved with a simple sed replacement:  

```shell
sed -i '' 's/threaded="1"/threaded="0"/g ~/.claws-mail/folderlist.xml
```

If folders are created on a regular basis, it might be worth running it as a periodic cronjob.  


_Thanks_  
_sko_

----
