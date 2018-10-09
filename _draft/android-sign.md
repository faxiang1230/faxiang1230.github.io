1.生成签名文件:
```
keytool -genkey -alias demo.keystore -keyalg RSA -validity 40000 -keystore demo.keystore
```
2.给应用签名:
```
#!/usr/bin/env sh
jarsigner -verbose -keystore ~/bin/demo.keystore -signedjar signed.apk $1 demo.keystore

./sign unsigned.apk
```
说明：
      1）jarsigner是工具名称，-verbose表示将签名过程中的详细信息打印出来，显示在dos窗口中；
      2）-keystore ~/bin/demo.keystore 表示签名所使用的数字证书所在位置
      3）-signedjar signed.apk notepad.apk 表示给传入的apk文件签名，签名后的文件名称为signed.apk；
      4）最后面的demo.keystore 表示证书的别名，对应于生成数字证书时-alias参数后面的名称
3.查看证书
查看apk的证书:
```
右键apk解压，目标文件是META-INF文件夹中的CERT.RSA文件，通过命令keytool.exe命令查看证书信息
可以查看签名的MD5、SHA1、SHA256值及签名算法
命令：keytool -printcert -file 目标文件路径
```
查看keystore的信息:
```
keytool -list -v -keystore 目标文件路径
```
