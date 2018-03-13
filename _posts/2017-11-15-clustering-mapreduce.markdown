---
layout:    post
title:      "mapreduce实现聚类分析搜狗数据"
subtitle:   "Analyse the log of sogou by using k-means algorithm base on mapreduce"
date:       2017-11-15 12:00:00
author:     "scyhssm"
header-img: "img/dog.jpeg"
header-mask: 0.3
catalog:    true
tags:
    - hadoop
    - mapreduce
    - 聚类
    - k-means
    - 数据分析
    - 搜狗搜索日志
---

> 对搜狗搜索词条中的结果在返回结果中的排名和用户点击的顺序号进行聚类算法。由于数据集大小达到4.6G+，选用mapreduce实现。后发现在测试状态下，4.6G+的大小过大了，运行要很长时间，电脑的配置不够。测试状态选择500MB的数据量运行聚类算法。

## 第一次编码解决问题
选用合适的聚类算法是成功的第一步，选择k-means算法对小内容进行聚类。  
第一遍运行将所有代码放在一个MyRunner类中，在代码量不大的情况下比较容易检查错误和实现思想。  
我们对 *该 URL 在返回结果中的排名* 和 *用户点击的顺序号* 同时聚类。  
去发现排名容易聚集在哪几个位置以及用户点击的顺序号容易聚集在哪几个位置，分别用x、y表示，发掘其中的关系。  
### k-means（K-平均算法）
k-平均算法源于信号处理中的一种向量量化方法，现在则更多地作为一种聚类分析方法流行于数据挖掘领域。  
k-平均聚类的目的是：把n个点（可以是样本的一次观察或一个实例）划分到k个聚类中，使得每个点都属于离他最近的均值（此即聚类中心）对应的聚类，以之作为聚类的标准。  
这个问题将归结为一个把数据空间划分为Voronoi cells的问题。  
这个问题在计算上是困难的（NP困难），不过存在高效的启发式算法。一般情况下，都使用效率比较高的启发式算法，它们能够快速收敛于一个局部最优解。  
这些算法通常类似于通过迭代优化方法处理高斯混合分布的最大期望算法（EM算法）。  
而且，它们都使用聚类中心来为数据建模；然而k-平均聚类倾向于在可比较的空间范围内寻找聚类，期望-最大化技术却允许聚类有不同的形状。
k-平均聚类与k-近邻之间没有任何关系（后者是另一流行的机器学习技术）。
### 准备
创建一个Centers类的点，表示聚类中心。  
内含3个属性，属性x表示用户点击的URL在返回结果中的排名，属性y表示用户点击的顺序号。
```
public double x;
public double y;
public int num;
```
创建setter和getter,Centers类代码如下
```
public class Centers{
    public double x;
    public double y;
    public int num;

    public double getX() {
        return x;
    }

    public void setX(double x) {
        this.x = x;
    }

    public double getY() {
        return y;
    }

    public void setY(double y) {
        this.y = y;
    }

    public int getNum() {
        return num;
    }

    public void setNum(int num) {
        this.num = num;
    }
}
```
新建一个class，将主函数，mapper和reducer代码放进去，我们先迭代一次，看实验结果如何。  
在该class内定义一个static类，放5个中心，先选用5个中心运行聚类算法，查看聚类效果。  
```
static Centers[] centers = new Centers[5];
```
### 实现map
首先是map过程，先确定map并行度，恰当的map并行度大概是每个节点10-100个map。  
map过程计算和各个中心的距离，用欧几里得计算距离（即两点间的直线距离），输出key为中心序号，value为该节点的x，y值。  
首先令我们自定义的Mapper类继承org.apache.hadoop.mapreduce.Mapper。  
<LongWritable,Text,IntWritable,Text>前两个为keyin和valuein，mapper初始化可以不用管keyin，输入是valuein。后两个为keyout和valueout，是输入到reducer的内容。
```
public static class MyMapper extends org.apache.hadoop.mapreduce.Mapper<LongWritable,Text,IntWritable,Text>{

}
```
重写mapper内的map()方法，LongWriteble key表示第一个输入参数，Text value表示第二个参数。  
Context context是上下文对象，是mapper的内部类。为了在map和reduce任务中跟踪task状态，自然的MapContext记录了map执行的上下文，在mapper类中，这个context可以存储job conf的信息。  
context同时也是map和reduce执行中各个函数的桥梁。
```
protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
}
```
第一行获得传过来的长串String。第二行分割字符串，行中列分割使用\t，换成空格就会报错。  
第三行定义了tmp存储比较值。第四行和第五行从String中获得内容，注意要转换成double类型。
```
String line = value.toString();
String[] words = line.split("\t");
double tmp;
double x = Double.parseDouble(words[3]);
double y = Double.parseDouble(words[4]);
```
比较函数用于选择在诸多点中距离最短的中心点，首先计算第一条记录和第一个点的距离，将其设为最小值，同时距离最短点则为minzb。  
接下来对4个点比较距离大小，保存最小距离的点。
```
double min = (x-centers[0].x)*(x-centers[0].x)+(y-centers[0].y)*(y-centers[0].y);
int minzb=0;
for(int i=1;i<5;i++)
{
    tmp = (x-centers[i].x)*(x-centers[i].x)+(y-centers[i].y)*(y-centers[i].y);
    if (tmp < min)
    {
        minzb = i;
        min = tmp;
    }
}
```
输出一次运行的结果到reduce，keyout是中心点，valueout是将double类型的输入点x，y转化为String。
```
context.write(new IntWritable(minzb),new Text(Double.toString(x)+" "+Double.toString(y)));
```
### 实现reduce
下一步是reducer，现在有5个key，每个key对应一个reduce任务。  
因为是测试任务，所以输入的行数较少，5个reduce足够用。  
如果内容更多，我们只有5个reduce任务显然是不够的，可以考虑在map后本地先用combine预处理，得到第一次的处理结果，再将处理结果传送给reduce任务。  
5个reduce分别形成5个key，每个key对应一串输入的内容，对该串内容处理，最后输出。  
首先声明一个静态类MyReducer，该类继承自org.apache.hadoop.mapreduce.Reducer<IntWritable,Text,IntWritable,Text>。  
注意不能继承旧类，旧的类虽然可以使用，但是语法和现在主流表述差别很大，且表达繁琐，不推荐使用。  
keyin的类型为IntWritable，是hadoop对int类型的序列化对象，实现了序列化接口。我们自己也可以做自己的序列化对象，这就是比较高端的功能了。  
keyin是从map传来的key值序列化对象，valuein是从map输出的String值序列化对象。  
keyout保持传入的key值不改变输出，valueout则是经过计算得到的聚类中心点，一共有5个。
```
public static class MyReducer extends org.apache.hadoop.mapreduce.Reducer<IntWritable,Text,IntWritable,Text>{
}
```
重用reduce，和map一样有key，value，context。注意，这里的key和map中的key不同，具有含义了。  
value则是一串value的列表，接受自map传来的大量输出值。context和map功能相同。
```
@Override
protected void reduce(IntWritable key,Iterable<Text> values,Context context)throws IOException,InterruptedException{
}
```
从key中得到中心点，创建sumx，sumy。sumx用于统计传入x值的总和，而sumy用于统计传入y值的总和。  
num用于计算传入点的个数，sx是聚类中心含有点中x值的总和，sy是聚类中心含有点中y值的总和。
```
int n = key.get();
double sumx=0,sumy=0;
int num=0;
double sx = centers[n].x * centers[n].num;
double sy = centers[n].y * centers[n].num;
```
这部分是对得到的value值进行分割。首先得到value值，因为传送过来是用空格分隔的，所以split中用空格得到两个word。  
将得到的两个word转化为xx和yy，即传送过来点的xx和yy。  
计算总和的sumx和sumy分别加上xx和yy值，最后将点的个数加1。
```
for(Text value:values){
    String[] word = value.toString().split(" ");
    double xx = Double.parseDouble(word[0]);
    double yy = Double.parseDouble(word[1]);
    sumx+=xx;
    sumy+=yy;
    num++;
}
centers[n].x = (sumx+sx)/(num+centers[n].num);
centers[n].y = (sumy+sy)/(num+centers[n].num);
centers[n].num = num+centers[n].num;
```
输出中心位置和该聚类中心拥有点的总和，输出到结果中去。我们可以查看执行的结果和控制台输出。  
```
System.out.println(centers[n].x+" "+centers[n].y+" "+centers[n].num);
context.write(key,new Text(Double.toString(centers[n].x)+" "+Double.toString(centers[n].y)));
```
控制台输出
```
7.963910576884096 6.137443373113865 68
9.867006967626644 7.645824740207708 27
7.509633716676818 1.7998523311814156 1677
1.4478611300268893 1.2481601023363167 6268
4.094127095794054 1.9761416229390574 1965
```
hadoop fs -ls /tmp ,可以看到有两个文件，一个文件是_SUCCESS,表示执行成功。另一个文件是part-r-00000,是执行结果保存位置。  
hadoop fs -cat /tmp/part-r-00000,5个中心，5组数据，和上面内容基本相同
```
0	7.963910576884096 6.137443373113865 68
1	9.867006967626644 7.645824740207708 27
2	7.509633716676818 1.7998523311814156 1677
3	1.4478611300268893 1.2481601023363167 6268
4	4.094127095794054 1.9761416229390574 1965
```
### 运行程序
mapper和reducer类是定义在运行类中的，我们要在运行类中写入main方法运行程序。要throws Exception抛出异常，否则程序代码会报错。  
```
public static void main(String[] args) throws Exception{
}
```
先用随机数获得几个初始化随机的中心点。
```
Random r = new Random();
for (int i=0;i<5;i++)
{
    centers[i] = new Centers();
    centers[i].x = r.nextDouble()*10+1;
    centers[i].y = r.nextDouble()*10+1;
    centers[i].num = 1;
}
```
新建一个配置文件管理配置信息
```
Configuration conf = new Configuration();
```
从conf中获得job实例，一个job管理多个task，并跟踪task
```
Job job = Job.getInstance(conf);
```
设置运行类，MyRunner。设置mapper类，MyMapper。设置reducer类，MyReducer。  
设置输入的key和value以及输出的key和value。
```
job.setJarByClass(MyRunner.class);
job.setMapperClass(MyMapper.class);
job.setReducerClass(MyReducer.class);

job.setMapOutputKeyClass(IntWritable.class);
job.setMapOutputValueClass(Text.class);
job.setOutputKeyClass(IntWritable.class);
job.setOutputValueClass(Text.class);
```
设置读取的文件路径
```
FileInputFormat.setInputPaths(job,"hdfs://hdp-01:9000/sogou_full/sogou.10k.utf8");
```
需要先检查是否已经存在输出文件夹，有的话要删除，否则会报文件夹已存在的错误，再为job设置输出文件路径
```
Path path = new Path("hdfs://hdp-01:9000/tmp/");
FileSystem fileSystem = path.getFileSystem(conf);
if(fileSystem.exists(path)){
    fileSystem.delete(path,true);
}
FileOutputFormat.setOutputPath(job,path);
```
提交作业,等待作业的完成，成功或者失败值
```
boolean res = job.waitForCompletion(true);
```
系统退出,退出代码0或者1，1的话执行失败，0执行成功
```
System.exit(res?0:1);
```
程序代码如下,注意将导入的包也写入，以防导入错误包，可以和我比对。
```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.util.Random;

public class MyRunner {
    static Centers[] centers = new Centers[5];

    public static class MyMapper extends org.apache.hadoop.mapreduce.Mapper<LongWritable,Text,IntWritable,Text>{

        @Override
        protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
            String line = value.toString();
            String[] words = line.split("\t");
            double tmp;
            double x = Double.parseDouble(words[3]);
            double y = Double.parseDouble(words[4]);
            double min = (x-centers[0].x)*(x-centers[0].x)+(y-centers[0].y)*(y-centers[0].y);
            int minzb=0;
            for(int i=1;i<5;i++)
            {
                tmp = (x-centers[i].x)*(x-centers[i].x)+(y-centers[i].y)*(y-centers[i].y);
                if (tmp < min)
                {
                    minzb = i;
                    min = tmp;
                }
            }
            context.write(new IntWritable(minzb),new Text(Double.toString(x)+" "+Double.toString(y)));
        }
    }

    public static class MyReducer extends org.apache.hadoop.mapreduce.Reducer<IntWritable,Text,IntWritable,Text>{

        @Override
        protected void reduce(IntWritable key,Iterable<Text> values,Context context)throws IOException,InterruptedException{
            int n = key.get();
            double sumx=0,sumy=0;
            int num=0;
            double sx = centers[n].x * centers[n].num;
            double sy = centers[n].y * centers[n].num;
            for(Text value:values){
                String[] word = value.toString().split(" ");
                double xx = Double.parseDouble(word[0]);
                double yy = Double.parseDouble(word[1]);
                sumx+=xx;
                sumy+=yy;
                num++;
            }
            centers[n].x = (sumx+sx)/(num+centers[n].num);
            centers[n].y = (sumy+sy)/(num+centers[n].num);
            centers[n].num = num+centers[n].num;
            System.out.println(centers[n].x+" "+centers[n].y+" "+centers[n].num);
            context.write(key,new Text(Double.toString(centers[n].x)+" "+Double.toString(centers[n].y)+" "+centers[n].num));
        }
    }
    public static void main(String[] args) throws Exception{
        Random r = new Random();
        for (int i=0;i<5;i++)
        {
            centers[i] = new Centers();
            centers[i].x = r.nextDouble()*10+1;
            centers[i].y = r.nextDouble()*10+1;
            centers[i].num = 1;
        }
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        job.setJarByClass(MyRunner.class);
        job.setMapperClass(MyMapper.class);
        job.setReducerClass(MyReducer.class);

        job.setMapOutputKeyClass(IntWritable.class);
        job.setMapOutputValueClass(Text.class);
        job.setOutputKeyClass(IntWritable.class);
        job.setOutputValueClass(Text.class);

        FileInputFormat.setInputPaths(job,"hdfs://hdp-01:9000/sogou_full/sogou.10k.utf8");
        Path path = new Path("hdfs://hdp-01:9000/tmp/");
        FileSystem fileSystem = path.getFileSystem(conf);
        if(fileSystem.exists(path)){
            fileSystem.delete(path,true);
        }
        FileOutputFormat.setOutputPath(job,path);
        boolean res = job.waitForCompletion(true); //true print information about job
        System.exit(res?0:1);
    }
}
```
实际运行程序3次，来得到聚类中心，观察3次结果的差别
第一次结果
```
7.565554260367236 1.9190270118409338 1453
2.0526206913144667 1.3403351735401978 8252
9.790181898750935 3.746865721477528 134
7.547317689916435 5.640447076030208 147
9.23537548457083 8.468984525819067 19
```
第二次结果
```
2.7259857279959023 1.288383322597096 9091
9.30641427152966 3.932783218409644 373
7.640850929589265 7.773732872655366 14
4.110638415554321 3.8667988910433344 522
2.6778446533430977 8.16779167964483 5
```
第三次结果
```
5.193338189213054 4.935468694892412 136
9.789185987816927 3.8467898674029533 247
2.022874468366132 1.303797940036555 8161
7.496603809472903 1.7429588419004785 1280
7.144220219083458 4.664608804602883 181
```
第一次结果有2.05，1.34聚类中心，第二次有2.72,1.28聚类中心，第三次有2.02，1.30聚类中心。  
这三个中心是大量数据的落点位置，实验证明，他们的相差不大，说明绝大部分的人习惯于第一次点击点击的url是搜索结果中的其二个，这是很有趣的现象。  
另外有数据是第一次结果7.56，1.91，第二次9.30.3.93，第三次7.49，1.74，这又是一个比较有趣的现象，大多数人第二次习惯找比较靠后的搜索条目。  
但是我们的数据量不大，为了演示，用的是1w的数据量进行的测试，因此聚类可能还不够精准。尤其是在落点较少的位置，出现了较大的震荡。  
如第二次，2.67和8.16上只有落下了5个点，我们增加数据量到500w，运行测试，由于担心运行速度过慢，我们加入了Combiner。  
Combiner在我们的程序中直接使用reducer代码，其实际意义就相当于在本地运行一次reduce。  
该combine将数据进行预处理，再分发给真正的reduce。k-means的k只有5个，数据量增大reduce负担加重。  
加入combiner能在节点中预先使用reduce，相当于一次外排序归并算法。
```
job.setMapperClass(MyMapper.class);
job.setCombinerClass(MyReducer.class);    //加入这句
job.setReducerClass(MyReducer.class);
```
在数据量不够的情况下。如上一次，还会出现有些reduce根本没有value进入的情况，而在数据量够大的情况下，一定会有点被选到某个聚类中心。  
调整k值为4，考虑到5可能过大，会有很多相似点出现，调整k值，第一次输出结果,收敛情况如下。
```
7.062163039444282 6.765846888575095 10408
8.911026592799185 2.7049637453747932 419075
2.3809391525100434 1.3947193467553314 4560831
4.74050417557854 6.317502674728926 9690
```
第二次输出结果，收敛情况
```
4.147003324492515 9.22514421753456 314
2.888032044962833 1.4579007459899211 4924577
9.311834173997815 6.4378538828999865 40654
3.2121983490229944 5.285486485869617 34459
```
上面这个情况能够找到3个位置相近点，最后一个点问题就很大了。
```
2.38 1.39
2.88 1.45
```
```
4.74 6.31
3.21 5.28
```
```
7.06 6.76
9.31 6.43
```
```
8.91 2.70
4.14 9.22
```
我们可以从前3条中看到这几个聚类，第一个聚类是两个值都较小的，点击的位置在2，1左右，第二个聚类是两个值都较大的，点击的位置在8，6左右，第三个聚类是在中间位置。  
由于数据的说服力还不足，我们改进算法成多次迭代来分析。  
说明一次迭代的聚类有明显的短板，我们改成多次迭代。  
## 第二次编码解决问题
由于上一次编码只有一次迭代，严格意义上并不能算作是k-means算法，但是基本的思想我们已经理清，现在需要实现多次迭代的k-means算法，获得更加严谨的数据。  
### 准备
我们在之前的包内编写新的类，因为还得用到之前的Centers类。  
新建一个NewRunner类，该类中含有3个数组，一个是boolean数组，一个是incenters数组，一个是outcenters数组。  
另外根据单次迭代的经验，设置k为4分辨较高，如果再有多的点，分出来的类会比较接近。
```
static Centers[] incenters = new Centers[4];
static Centers[] outcenters = new Centers[4];
static boolean[] flag={false,false,false,false};
```
### 实现newmap
新的map和之前的map处理方法类似，但是不同的是centers数组改为了incenters数组。  
修改的原因是我们需要outcenters数组保存输出数据，用以和incenters比较。  
如果像第一次一样直接覆盖前一次记录，那么无法获得收敛效果，就不知道何时结束k-means聚类算法。  
在前一次代码的基础上微调,贴出改动部分
```
double min = (x-incenters[0].x)*(x-incenters[0].x)+(y-incenters[0].y)*(y-incenters[0].y);
tmp = (x-incenters[i].x)*(x-incenters[i].x)+(y-incenters[i].y)*(y-incenters[i].y);
```
### 实现newreduce
在这里，reduce承担的任务又加重了，我们之前的计算新的聚类中心方法从centers改为了outcenters
```
outcenters[n].x = (sumx+sx)/(num+incenters[n].num);
outcenters[n].y = (sumy+sy)/(num+incenters[n].num);
outcenters[n].num = num+incenters[n].num;
```
输出新的聚类中心
```
System.out.println(outcenters[n].x+" "+outcenters[n].y+" "+outcenters[n].num);
```
判断如果新迭代的聚类中心和上一次迭代的聚类中心相差距离不超过0.1，则进入if条件中。  
将n号的标记打为true，表名第n个点已经收敛完成。  
如果还没有完全收敛，则将上一次迭代的incenters数组中的内容用新的内容替换，为下一次迭代做准备。  
```
if (Math.abs(incenters[n].x-outcenters[n].x)<0.1&&Math.abs(incenters[n].y-outcenters[n].y)<0.1)
{
    flag[n]=true;
    context.write(key,new Text(Double.toString(outcenters[n].x)+" "+Double.toString(outcenters[n].y)+" "+outcenters[n].num));
}
else
{
    incenters[n].x=outcenters[n].x;
    incenters[n].y=outcenters[n].y;
    incenters[n].num = outcenters[n].num;
    context.write(key,new Text(Double.toString(outcenters[n].x)+" "+Double.toString(outcenters[n].y)+" "+outcenters[n].num));
}
```
### 运行程序
我们为了能够确定迭代的次数，新加入了count，来累计迭代次数。
```
int count=0;
```
初始化outcenters和incenters，注意虽然我们目前不用outcenters，但是要先为他赋予空间。  
我遇到这个问题调试了一会才找到原因。
```
for (int i=0;i<4;i++) {
    outcenters[i] = new Centers();
    incenters[i] = new Centers();
    incenters[i].x = r.nextDouble() * 10 + 1;
    incenters[i].y = r.nextDouble() * 10 + 1;
    incenters[i].num = 1;
}
```
无限迭代，直到满足收敛条件停止迭代
```
while(true){

}
```
每次迭代后，count要+1
```
count++;
```
下面的flag必须重新初始化赋值为false，如果不修改回来，那么迭代只用几次就会返回。  
另外，incenters数组内对象的num属性必须初始化为1，否则会累加总和，多次迭代后总量会变得非常大。
```
count++;
for(int i=0;i<4;i++)
{
    flag[i]=false;
    incenters[i].num=1;
}
```
job设置class，因为runner，mapper和reducer类都改变了
```
job.setJarByClass(NewRunner.class);
job.setMapperClass(NewMapper.class);
job.setReducerClass(NewReducer.class);
```
修改系统退出条件，当4个flag都为true时，算法已经收敛到我们要求内，可以退出，并输出迭代次数。
```
if(flag[1]&&flag[2]&&flag[3]&&flag[0])
{
    System.out.println(count);
    System.exit(0);
}
```
贴出新的k-means算法代码
```
package SOGOU;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.*;
import java.util.Random;

public class NewRunner {
    static Centers[] incenters = new Centers[4];
    static Centers[] outcenters = new Centers[4];
    static boolean[] flag={false,false,false,false};
    public static class NewMapper extends org.apache.hadoop.mapreduce.Mapper<LongWritable,Text,IntWritable,Text>{

        @Override
        protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
            String line = value.toString();
            String[] words = line.split("\t");
            double tmp;
            double x = Double.parseDouble(words[3]);
            double y = Double.parseDouble(words[4]);
            double min = (x-incenters[0].x)*(x-incenters[0].x)+(y-incenters[0].y)*(y-incenters[0].y);
            int minzb=0;
            for(int i=1;i<4;i++)
            {
                tmp = (x-incenters[i].x)*(x-incenters[i].x)+(y-incenters[i].y)*(y-incenters[i].y);
                if (tmp < min)
                {
                    minzb = i;
                    min = tmp;
                }
            }
            context.write(new IntWritable(minzb),new Text(Double.toString(x)+" "+Double.toString(y)));
        }
    }

    public static class NewReducer extends org.apache.hadoop.mapreduce.Reducer<IntWritable,Text,IntWritable,Text>{

        @Override
        protected void reduce(IntWritable key,Iterable<Text> values,Context context)throws IOException,InterruptedException{
            int n = key.get();
            double sumx=0,sumy=0;
            int num=0;
            double sx = incenters[n].x * incenters[n].num;
            double sy = incenters[n].y * incenters[n].num;
            for(Text value:values){
                String[] word = value.toString().split(" ");
                double xx = Double.parseDouble(word[0]);
                double yy = Double.parseDouble(word[1]);
                sumx+=xx;
                sumy+=yy;
                num++;
            }
            outcenters[n].x = (sumx+sx)/(num+incenters[n].num);
            outcenters[n].y = (sumy+sy)/(num+incenters[n].num);
            outcenters[n].num = num+incenters[n].num;
            System.out.println(outcenters[n].x+" "+outcenters[n].y+" "+outcenters[n].num);
            if (Math.abs(incenters[n].x-outcenters[n].x)<0.1&&Math.abs(incenters[n].y-outcenters[n].y)<0.1)
            {
                flag[n]=true;
                context.write(key,new Text(Double.toString(outcenters[n].x)+" "+Double.toString(outcenters[n].y)+" "+outcenters[n].num));
            }
            else
            {
                incenters[n].x=outcenters[n].x;
                incenters[n].y=outcenters[n].y;
                incenters[n].num = outcenters[n].num;
                context.write(key,new Text(Double.toString(outcenters[n].x)+" "+Double.toString(outcenters[n].y)+" "+outcenters[n].num));
            }
        }
    }
    public static void main(String args[]) throws Exception{
        int count=0;
        Random r = new Random();
        for (int i=0;i<4;i++) {
            outcenters[i] = new Centers();
            incenters[i] = new Centers();
            incenters[i].x = r.nextDouble() * 10 + 1;
            incenters[i].y = r.nextDouble() * 10 + 1;
            incenters[i].num = 1;
        }
        while(true){
            count++;
            for(int i=0;i<4;i++)
            {
                flag[i]=false;
                incenters[i].num=1;
            }
            Configuration conf = new Configuration();
            Job job = Job.getInstance(conf);
            job.setJarByClass(NewRunner.class);
            job.setMapperClass(NewMapper.class);
            job.setReducerClass(NewReducer.class);

            job.setMapOutputKeyClass(IntWritable.class);
            job.setMapOutputValueClass(Text.class);
            job.setOutputKeyClass(IntWritable.class);
            job.setOutputValueClass(Text.class);
            FileInputFormat.setInputPaths(job,"hdfs://hdp-01:9000/sogou_full/sogou.500w.utf8.ext");
            Path path = new Path("hdfs://hdp-01:9000/tmp/");
            FileSystem fileSystem = path.getFileSystem(conf);
            if(fileSystem.exists(path)){
                fileSystem.delete(path,true);
            }
            FileOutputFormat.setOutputPath(job,path);
            boolean res = job.waitForCompletion(true); //true print information about job
            if(flag[1]&&flag[2]&&flag[3]&&flag[0])
            {
                System.out.println(count);
                System.exit(0);
            }
        }
    }
}
```
运行程序3次,得到3组数据，虽然可以分出大概的范围，但是仍然不够精确。  
但是和上一次结果有着很大的不同，这也可以看出，通过一次迭代不能准确找到聚类中心。  
如果样本充足，或许可以能够通过一次迭代找到聚类中心的，但是在样本不充足的情况下，还是老老实实多写一点代码，找到聚类中心。
```
1.4650892982120827 1.1361333239069205 3283106
8.277649601399263 4.553432254019519 227111
7.4979029058233815 1.7114316450620564 583201
4.026218210119877 2.0560543805666227 906586
7
8.84999009277398 3.1084157805844637 404914
6.236955671192182 1.621780977346147 511181
1.4651997011358089 1.134978989654943 3282327
3.907177256904062 2.262078372389326 801582
13
4.211456065667271 3.6501026081913888 288477
1.5391214076563071 1.174140543759781 3445557
8.523428068124636 2.7533200203492787 526805
5.011954045381209 1.4574248745882672 739165
7
```
我们分别将k值减小1和让k值增大一，来看看在这两种情况下三次运行程序结果有何不同。  
修改部分代码，通过修改代码，我们发现代码写得还是不够漂亮。漂亮的代码应该能够通过只单一修改一个变量k而达到目的。  
我们修改了超过5处位置，如果代码行数再增加，需要修改的地方更多。如果出现一些问题，导致程序错误，找BUG花费大量时间，得不偿失。  
希望能够引以为戒，养成良好的习惯，不要因为代码量不大就偷懒不考虑设计模式，好的设计模式能够大大增加开发效率。  
k为3，输出结果，我们可以看到，当k为3时，迭代次数明显增多而且单次job执行时间相对k为4时更久。  
可能原因是少了一个reduce任务，其他reduce分到的任务量增加了，增加了处理时间。  
另外因为数据量增加，收敛更加精确，迭代次数明显增加,我们甚至看到，第二次和第三次的结果，除了迭代次数不同，其他数据完全相同。
```
8.489825439699082 2.8109296161488815 535061
4.934038379417361 1.93571087736838 943427
1.566201072627069 1.2199451739404403 3521515
8
4.668180683897276 2.072949139356176 947812
1.5396084631541653 1.185277860101063 3456561
8.33834987886804 2.6266719719829976 595630
13
4.668180683897276 2.072949139356176 947812
1.5396084631541653 1.185277860101063 3456561
8.33834987886804 2.6266719719829976 595630
14
```
我们用图来表示，可以更直观地看出来。  
![ k为3的聚类中心 ](/img/k为3散点图.png)
k为5，输出结果，我们可以从这三次测验看出，k为5的效果和k为4类似，效果不够理想，3次聚类中心差别都很大。  
差别是否大其实可从点总数的差距上看出。像k为3时，一个聚类中心附近的点数量和其他运行结果差别不会很大，但是在k为5时，点的数量就参差不齐了。  
```
1.4650893206376718 1.136133336584487 3283106
6.771591531612687 1.7160559064917444 412547
7.436166526402943 5.158104796792856 161609
9.33622600823018 2.226819901712611 247638
4.016841720612552 2.014558168101048 895105
11
1.4625592663947347 1.114511789173148 3251885
3.245983463880691 2.8779949774417686 393747
7.957426547201674 4.9539692288100845 180489
8.317657645873517 1.8209040897394957 447722
4.845329110283181 1.5966418834319624 726162
8
6.771591531610803 1.716055906493494 412547
1.302154548325988 1.1512138786947954 2968419
9.33622600822881 2.226819901714297 247638
7.436166526408367 5.158104796807297 161609
3.7528746694842137 1.749062442515367 1209792
7
```
用二维坐标系表示k为5时的聚类分布,这张散点图的点分布明显相比k为3时更乱。  
![ k为5的聚类中心 ](/img/k为5散点图.png)
如果我们对收敛的要求更高，从0.1改为0.05，应该可以获得更精确的结果。  
```
8.317540581460618 1.8208797540532 447717
4.799479103109028 1.9234878939703106 860440
8.197595230546957 5.085023257748928 160263
2.6585740987576516 1.8095673060670314 827674
1.2371406599011634 1.051559773040883 2703911
8
1.5391214076563071 1.174140543759781 3445557
4.834485726243702 1.4982581478009336 678589
8.374472449706085 5.939398004645684 82226
8.329123234801587 2.1016086068451765 513924
4.147157035337348 3.559322578439502 279709
8
1.3035167895415203 1.1560104136338598 2973764
5.669215678583732 3.8111429356148205 254871
7.643597973006236 1.5302787605689132 511300
3.6807072186446694 1.5699700426210585 1104824
9.126915520638812 4.510638023782632 155246
10
```
得到二维坐标系散点图,我们看到，当收敛参数改为0.05时，我们得到了聚类更加明显的二维坐标系。
![ 更精确的k为5的聚类中心 ](/img/更精确的k为5.png)
增加k为3，收敛参数为0.1的坐标
```
4.668180683894706 2.0729491392943316 947812
1.5396084633966458 1.1852778607771406 3456561
8.33834987880211 2.626671972128559 595630
11
4.675483025171712 2.0959636445497614 953196
8.359511631708973 2.595012095734688 590288
1.5396150114145823 1.1851828920574061 3456519
7
```
基本相比上一张k为3的图没有大的变化。
![ 扩展k为3 ](/img/扩展k为3.png)
## 分析并总结
可以看到k-means算法的特性，如我们在选择聚类个数即k时，k是一个输入参数，不恰当的k值会造成糟糕的结果，如k为5时程序执行最后得到的聚类中心相差非常大。  
因此需要特征检查来决定数据集的聚类数目。  
另外，在数据量不大的情况下，聚类结果的和初始聚类中心的相关性非常高。  
调整收敛参数来适应不同的k值，我们希望数据越多越好，这样就能得到更精确的结果。k等于5时结果不精确的一个原因是收敛参数设置还是太大，另外是数据量仍然不够多，导致结果不理想。  
分析k为3时的聚类中心图，可以发现，第一次点击，基本大多数人都偏好于点击第一个搜索条目。第二次点击，大多数人会偏向于点击第四个或者第五个搜索条目。第三次点击，大多数人倾向于点击9和10的搜索条目。  
用户对得到结果的选择总是跳跃性的，或者说大多数人呈跳跃性。这是一个很有趣的现象，如果搜狗想要坏一点，可以把广告等内容放在中间4-5位。这样既没有影响到搜索结果的正确率，第一位还是最有可能是用户需要的内容，又可以增加广告收入。  
另外我们可以看到，聚类在9以外的中心点是不存在的，虽然会有用户找在第一页以外的搜索条目，但是绝大多数用户还是选择第一页的搜索条目。  
一个简单的聚类算法就写到这里，希望你们能够在这个基础上发掘更有价值的内容。

## 最后
通过后续的考虑，发现实验有不足的地方，由于不能够保证取到的点分布均匀，让（1，1），（2，1）这类点都聚到一起，最后出现问题。即结论的正确性，应该要找到合适的初始化点，来使结果更正确。

由于发现结果的错误性，比如说(1,2)，(2,2),(2,1)都更加靠近(1,1)，聚类出的第二次点击更加可能的位置是第五个结论是不正确的。

这个实验只能作为是一个Hadoop下MapReduce的实践，而没有实际的作用，结果是错误的。

首先我们需要弄清聚类的目的，是将数据聚类后得出聚类的中心，用以分类样本。

假设以原来实验的做法做聚类，随机出的聚类中心大约为（1，1），（5，2），（9，3）。即我将(2,2)也归到（1，1）这个中心去了，这是错误的。

（2，2）应该和（1，1）严格区分，按照我们最后需要分析的结果，其实应该需要有10*10个类，（5，2）和（4，2）也应该严格区分，划为不同的类。

这个实验其实是错误的，属于没有理清需求，选择了错误的方法，但实现MapReduce的过程还是有所学习到的。

如果说要做对用户更喜爱点击哪个位置去考虑，应该group by x,y ，直接计算（x，y）最终有多少个值，按照点击次数得到用户偏好点击位置。

或者固定第一次点击，group by x，这个在日志分析的Hive实验就有涉及。

要用一种算法，不能盲目使用，要充分了解需求和算法能够解决的问题后再去实践，否则就像这个MapReduce任务一样，吃力不讨好。
