# java 调用linux 脚本并获取返回值

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;今天分享下java 调用 shell脚本 并获取返回值。



## 代码实践

```
String cmd = "df -h";
StringBuffer sb = new StringBuffer();
Process process = Runtime.getRuntime().exec(cmd);
BufferedReader br = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = br.readLine()) != null) {
    sb.append(line).append("\n");
}
String result = sb.toString();
```

这个result就是执行cmd后的结果。



## &&或者管道 | 导致无法获取结果

```
String cmd = "cd /root && df -h";
StringBuffer sb = new StringBuffer();
Process process = Runtime.getRuntime().exec(cmd);
BufferedReader br = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = br.readLine()) != null) {
    sb.append(line).append("\n");
}
String result = sb.toString();
```

脚本改成 包含&&的话，就发现result是个空值。 因为 shell脚本中如果有多个命令，那么在java中使用BufferedReader获取脚本的输出时，只能获取到第一个命令的输出，使用/bin/sh -c则能获取到所有的echo输出。

```
Process process = Runtime.getRuntime().exec(cmd);
// 修改为
Process process = Runtime.getRuntime().exec(new String[]{"sh", "-c", cmd});
```

同样的情况如果执行带 sudo 的命令，而报权限问题，很有可能也是这个问题。

比如这个shell，sudo只能覆盖到 && 之前

```
sudo cd /data/aaa && nohup java -jar test.jar
```

利用 "*sh* -*c*" 命令,它可以让 bash 将一个字串作为完整的命令来执行

```
sh -c sudo cd /data/aaa && nohup java -jar test.jar
```



