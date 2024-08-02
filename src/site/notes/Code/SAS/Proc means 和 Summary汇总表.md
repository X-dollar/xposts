---
{"dg-publish":true,"permalink":"/code/sas/proc-means-summary/","created":"2024-05-27T17:51:33.042+08:00","updated":"2024-08-02T17:17:44.435+08:00"}
---


# Summary的类型
## 频数汇总表
### cat项的增减需求
1. **添加cat项: 使用 `format` + `preloadfmt`+ `proc means `**
	1. 对于自定义format,**未规定的部分保持原样**,也可使用 `other=` 对**未出现的情况进行统一归类**
	2. 对使用 `preloadfmt` 的cat单独使用 class,其他变量另一起一行进行class即可
2. **删除cat项**: 使用 `completetype` 后生成全内容,之后 `where`等方法进行**筛选输出**即可
```sas
* 数据集范例 ;
data cars1;
    length make type origin drivetrain $200;
    retain flag 1;
    set sashelp.cars;
    keep make type origin weight flag drivetrain;
    rename make=cat1 type=cat2 drivetrain=cat3;
    if _n_<=50;
run;

* 在原本cat上进行添加,只需要把添加部分的format写上,format-value未出现的值不会被转换 ;
proc format;
    value $cat1cc(notsorted multilabel)
    "Xiaomi"="Xiaomi"
    "BYD"="BYD"
    ;
    value $cat2cc(notsorted multilabel)
    "SUV","Sports"="SUV/Sports"
    "Heilicop"="Heilicop"
    ;
    invalue cat1cn(notsorted)
    "Xiaomi","BYD"=1
    "Audi","BMW"=2
    other=3
    ;
    invalue trtcn(notsorted)
    "Asia"=1
    "Europe"=2
    "USA"=3
    ;
quit;

data cars2;
    set cars1;
    trtn=input(origin,trtcn.);
    cat1n=input(cat1,cat1cn.);
proc sort;
by trtn cat1n cat1 cat2 cat3 ;
run;

proc means data = cars2 nway noprint completetypes;
    by trtn cat1n;
    class cat1 cat2/preloadfmt mlf order = formatted; 
	/* data:遵循format顺序 freq/unformatted等遵循频率/字母顺序*/
    class cat3;
    var flag ;
    output n=count out=means1(where=(cat2^="Sedan"));
    format cat1$cat1cc.cat2$cat2cc.;
quit;
```
[[Code/SAS/个人工具\|个人工具]]
### 呈现的处理顺序
**如果使用了format的顺序,则需要出表后再进行sort**
1. 特定的呈现顺序
	1. 有限表中使用catn
	2. 无限表中通常特定顺序只在表头,同样使用catn进行固定
2. 依照字符或汉字顺序的排序,直接使用 `sort` 即可,汉字需要额外option

# 使用 input 创建 dummy #dummy

1. 不单独定义变量长度则以默认长度为8,所以需要定义
2. `:` 用于将**原始数据以空格为分隔**符时,读取到空格符则停止读取
3. `&` **如果字符内容中含有空格**,则使用`&`并在cards下文的两个变量内容间使用两个空格
```sas
data dummy;
    input cat1: $200. cat2 & $200. cat3: $200. ;
    cards;
    Xiaomi Sport  All/Back
    Xiaomi S U V  All/Back
    ;
run;
```
# Proc means的控制语法
#### ways 和 types 控制输出哪种组合的 class 变量
```sas
proc means;
 class a b c d e;
 ways 2 3; 
 run;
* == ;
proc means;
 class a b c d e;
 types a*b a*c a*d a*e b*c b*d b*e c*d c*e d*e
       a*b*c a*b*d a*b*e a*c*d a*c*e a*d*e
       b*c*d b*c*e c*d*e;
 run;
```
`ways` 控制输出**多少个 `class` 变量组合的结果**
	**使用`nways` option,可以默认输出最高等级的ways值**
`types` 具体**控制输出哪些变量的组合结果**

#### `order=`控制语句
1. `data`:原先数据集的顺序(最常使用,搭配format顺序)
2. `unformat`:自行排序(可能会导致大写>小写)
3. `freq`:按照频率数进行排序
#### 控制proc means的计算值和输出数据集名
```sas
proc means data = temp1 noprint nway completetypes;
    class sex age/preloadfmt mlf order = data;
    var _value ;
    output n=count means=means out=temp_means1; * 输出个数和平均数,输出数据集为temp_means1 ;  
    format sex$f_sex. age f_age.;
quit;
```

### classdata=语句
**使用如下数据集**
```sas
data cars1;
    length make type origin $200;
    retain flag 1;
    set sashelp.cars;
    keep make type origin weight flag;
    if _n_<=50;
run;
```
##### classdata做额外补充dummy
option `classdata=` 数据集作为**dummy补充数据集**,要求变量长度和 `data=`一致
```sas
data dummy;
    length type origin $200;
    input type $1-8  origin $9-20;
    datalines;
Helicop China
;
run;

proc means data=cars1 classdata=dummy nway order=data noprint;
    class origin type;
    var flag;
    output n=count out=car_ouput;
run;
```
##### classdata+exclusive 只输出dummy中的存在的数据组合
```sas
data dummy;
    length type origin $200;
    input type $1-8  origin $9-20;
    datalines;
Helicop China
SUV     USA
;
run;

proc means data=cars1 classdata=dummy nway exclusive order=data noprint;
    class origin type;
    var flag;
    output n=count out=car_ouput;
run;
```

##### classdata+completetypes 输出包括dummy在内的所有变量组合
```sas
data dummy;
    length type origin $200;
    input type $1-8  origin $9-20;
    datalines;
Helicop China
;
run;

proc means data=cars1 classdata=dummy nway completetypes order=data noprint;
    class origin type;
    var flag;
    output n=count out=car_ouput;
run;
```

#### 使用 completetypes 输出元数据集中分类变量的所有组合
```sas
data dummy;
    length type origin $200;
    input type $1-8  origin $9-20;
    datalines;
Helicop China
;
run;

proc means data=cars1 classdata=dummy nway completetypes order=data noprint;
    class origin type;
    var flag;
    output n=count out=car_ouput;
run;
```
