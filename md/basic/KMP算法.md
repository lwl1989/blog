CreateTime:2015-08-23 16:22:30.0

### 起因

> 在网上找了一个KMP的PHP解决方案，http://blog.sina.com.cn/s/blog_65cbe2b10101eqxg.html ，    后面发现有bug,于是便自己来解决

```
<?php

function KMP($str) {
    $K = array(0);
    $M = 0;
    for($i=1; $i<strlen($str); $i++) {
        if ($str[$i] == $str[$M]) {
            $K[$i] = $K[$i-1] + 1;
            $M ++;
        } else {
            $M = 0;
            $K[$i] = $K[$M];
        }
    }
    return $K;
}

// KMP查找
function KMPMatch($src, $par, $debug = false) {
    $K = KMP($par);
    for($i=0,$j=0; $i<strlen($src); ) {
        if ($j == strlen($par)) return $i-$j;
        
    echo $i,"  ", $j, " ", $src[$i], $par[$j],  "<BR>";
        if ($par[$j] === $src[$i]) {
            $j++;
            $i++;
            //此处bug
        } else {
            if ($j === 0 && $par[$j] != $src[$i]) {
                $i++;
            }
            $j = $K[$j-1 >= 0 ? $j -1 : 0];
        }
    }
    //此处bug
    return false;
}
```

发生了生么情况呢？
   我用hello作为母串 去寻找llo的子串
始终是返回false
  
### 于是我进行了分析，发现问题出现在循环内：
1.$i++次数过多  按照kmp原理,跳过的次数是根据子串得出，所以，在比较相同时，不应该$i++,
因为循环体里已经++过了
2.验证循序错误
```
 if($index==$strlen) return $i-$index; //应该在循环末尾
```
### 改进后的代码如下
```
function kmpvalue($str){
    $key = array(0);  //init  key[0]=0;
    $M=0;   //keyvalue init 0
    for($i=1;$i<strlen($str);$i++){
        if($str[$i]==$str[$M]){
            $key[$i]=$key[$i-1]+1;
            $M++;
        }else{
            $M=0;
            $key[$i]=$key[$M];
        }
    }
    return $key;
}

function KMPsearch($src,$str){
    $key = kmpvalue($str); //0 1 0
    $strlen = strlen($str);
    for($i=0,$index=0;$i<strlen($src);$i++){   // h e l l o
        //if ($index == $strlen) return $i-$index;  //如果长度相等，直接相减判断
        if($src[$i]==$str[$index]){
            $index++;
        }else{
            if ($index === 0 && $str[$index] != $src[$i]) {  //如果j==0  且 str[0] != $src[1]  l!=e
                $i++;
            }
            $index = $key[$index-1>=0?$index-1:0];
        }
        if($index==$strlen) return $i-$index;
    }

    return false;
}
```
