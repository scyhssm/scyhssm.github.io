---
layout:    post
title:      "利用echarts进行数据分析"
subtitle:   "Analyse data by echarts"
date:       2017-12-06 12:00:00
author:     "scyhssm"
header-img: "img/main_echarts.png"
header-mask: 0.3
catalog:    true
tags:
    - echarts
    - 数据可视化
---

>echarts是一款使用方便的数据可视化工具，该作业目的是可视化全省GDP和财政收入这两个经济数据

## GDP可视化分析
首先对宁波市的市县区的GDP进行可视化分析，使用echarts中众多图表的折线图堆叠。首先考虑柱状图，柱状图的效果并不理想，到最后量会变得很小，数据就失去了意义。通过折线图我们能够了解GDP的变化量以及变化幅度。而且当数据被压缩得很小时，可视化结果并不美观。而堆叠图及能得到历年量的变化，也能得到总量的变化。
```
option = {
    title: {
        text: '折线图堆叠'
    },
    tooltip: {
        trigger: 'axis'
    },
    legend: {
        data:['慈溪市','宁海县','海曙区','江北区','北仑区','象山县','鄞州区','镇海区']
    },
    grid: {
        left: '3%',
        right: '4%',
        bottom: '3%',
        containLabel: true
    },
    toolbox: {
        feature: {
            saveAsImage: {}
        }
    },
    xAxis: {
        type: 'category',
        boundaryGap: false,
        data: ['2016','2015','2014','2013','2012','2011','2010','2009','2008','2007','2006','2005','2004','2003','2002']
    },
    yAxis: {
        type: 'value'
    },
    series: [
        {
            name: '慈溪市',
            type: 'line',
            stack: '总量',
            data: [1209, 1137, 1111, 1031,948, 876, 757,626,601,530,450,375,295,245,201]
        },
        {
            name: '宁海县',
            type: 'line',
            stack: '总量',
            data: [465, 434, 409, 384, 352, 323, 278,235,217,194,191,129,115,100,83]
        },
        {
            name: '海曙区',
            type: 'line',
            stack: '总量',
            data: [1377, 1310, 1292, 1228, 1163, 1092, 937,838,332,290,250,221,185,143,93]
        },
        {
            name: '江北区',
            type: 'line',
            stack: '总量',
            data: [383, 328, 302, 278, 254, 226, 191,159,156,137,120,101,47,59,50]
        },
        {
            name: '北仑区',
            type: 'line',
            stack: '总量',
            data: [1153, 1134, 991, 924, 671, 645, 548,446,422,377,318,274,200,165,111]
        },
        {
            name:'象山县',
            type:'line',
            stack:'总量',
            data: [437, 410, 388, 363, 336, 316, 270,237,220,192,173,153,132,109,89]
        },
        {
            name:'鄞州区',
            type:'line',
            stack:'总量',
            data: [1331, 1300, 1296, 1207, 1093, 940, 810,703,650,425,425,343,288,223,187]
        },
        {
            name:'镇海区',
            type:'line',
            stack:'总量',
            data: [753, 652, 645, 325, 305, 560, 450,358,152,243,210,189,145,162,109]
        }
    ]
};
```
生成的折线图堆叠，从该图我们可以看到近几年宁波的县区经济都在上涨，但是涨幅有所不同。涨幅较大的一年是08年，这一年GDP涨幅非常大，猜测有很大的可能是08年经融危机政府救市。
![宁波各县区折线图堆叠](/img/折线图堆叠.png)
以上是折线图堆叠在宁波各县区的应用，如果想要知道县区的GDP哪个较多哪个较少，GDP的涨幅，堆叠的折线图不利于分析。我们使用其他的图表来分析浙江省各市的数据。
```
app.title = '笛卡尔坐标系上的热力图';

var hours = ['2003', '2004', '2005', '2006', '2007', '2008', '2009',
        '2010', '2011','2012','2013','2014','2015','2016'];
var days = ['杭州市', '宁波市', '绍兴市',
        '温州市', '金华市', '嘉兴市'];

var data = [[0,0,4887],[0,1,5927],[0,2,6831],[0,3,8031],[0,4,9494],[0,5,11214],[0,6,11731],[0,7,13669],[0,8,16140],[0,9,18194],[0,10,19569],[0,11,21398],[0,12,23652],[0,13,26067]
,[1,0,1025],[1,1,1260],[1,2,1384],[1,3,1597],[1,4,1926],[1,5,2251],[1,6,2597],[1,7,3062],[1,8,3621],[1,9,3950],[1,10,4309],[1,11,4589],[1,12,4877],[1,13,5574]
,[2,0,1088],[2,1,1313],[2,2,1440],[2,3,1677],[2,4,1972],[2,5,2222],[2,6,2375],[2,7,2795],[2,8,3332],[2,9,3654],[2,10,3967],[2,11,4265],[2,12,4465],[2,13,4710]
,[3,0,1220],[3,1,1402],[3,2,1600],[3,3,1834],[3,4,2157],[3,5,2424],[3,6,5809],[3,7,2925],[3,8,3350],[3,9,3650],[3,10,4003],[3,11,4302],[3,12,4619],[3,13,5045]
,[4,0,756],[4,1,925],[4,2,1066],[4,3,1240],[4,4,1474],[4,5,1693],[4,6,1777],[4,7,2114],[4,8,2463],[4,9,2721],[4,10,2973],[4,11,3208],[4,12,3402],[4,13,3684]
,[5,0,823],[5,1,1002],[5,2,1158],[5,3,1345],[5,4,1586],[5,5,1819],[5,6,1919],[5,7,2315],[5,8,2703],[5,9,2914],[5,10,3163],[5,11,3352],[5,12,3517],[5,13,3760]]

data = data.map(function (item) {
    return [item[1], item[0], item[2] || '-'];
});

option = {
    tooltip: {
        position: 'top'
    },
    animation: false,
    grid: {
        height: '50%',
        y: '10%'
    },
    xAxis: {
        type: 'category',
        data: hours,
        splitArea: {
            show: true
        }
    },
    yAxis: {
        type: 'category',
        data: days,
        splitArea: {
            show: true
        }
    },
    visualMap: {
        min: 0,
        max: 27000,
        calculable: true,
        orient: 'horizontal',
        left: 'center',
        bottom: '15%'
    },
    series: [{
        name: 'Punch Card',
        type: 'heatmap',
        data: data,
        label: {
            normal: {
                show: true
            }
        },
        itemStyle: {
            emphasis: {
                shadowBlur: 10,
                shadowColor: 'rgba(0, 0, 0, 0.5)'
            }
        }
    }]
};
```
这是全省的热力图,清晰地显示了各市GDP的增加，利用颜色的深浅直观对比。
![笛卡尔积热力图](/img/GDP热力图.jpg)
## 财政收入可视化分析
宁波市各县区财政收入2016饼图
```
option = {
    backgroundColor: '#2c343c',

    title: {
        text: '2016宁波县区财政收入',
        left: 'center',
        top: 20,
        textStyle: {
            color: '#ccc'
        }
    },

    tooltip : {
        trigger: 'item',
        formatter: "{a} <br/>{b} : {c} ({d}%)"
    },

    visualMap: {
        show: false,
        min: 80,
        max: 600,
        inRange: {
            colorLightness: [0, 1]
        }
    },
    series : [
        {
            name:'访问来源',
            type:'pie',
            radius : '55%',
            center: ['50%', '50%'],
            data:[
                {value:95, name:'江北区'},
                {value:402, name:'北仑区'},
                {value:80, name:'宁海县'},
                {value:60, name:'象山县'},
                {value:338, name:'鄞州区'},
                {value:95,name:'镇海区'},
                {value:573,name:'海曙区'}
            ].sort(function (a, b) { return a.value - b.value; }),
            roseType: 'radius',
            label: {
                normal: {
                    textStyle: {
                        color: 'rgba(255, 255, 255, 0.3)'
                    }
                }
            },
            labelLine: {
                normal: {
                    lineStyle: {
                        color: 'rgba(255, 255, 255, 0.3)'
                    },
                    smooth: 0.2,
                    length: 10,
                    length2: 20
                }
            },
            itemStyle: {
                normal: {
                    color: '#c23531',
                    shadowBlur: 200,
                    shadowColor: 'rgba(0, 0, 0, 0.5)'
                }
            },

            animationType: 'scale',
            animationEasing: 'elasticOut',
            animationDelay: function (idx) {
                return Math.random() * 200;
            }
        }
    ]
};
```
饼图如下，可以看到宁波市最发达的城市核心地带是海曙区，其次是北仑和鄞州
![2016财政饼图](/img/2016财政饼图.jpg)
看看我们所在的鄞州区的财政收入和财政支出
```
app.title = '正负条形图';

option = {
    tooltip : {
        trigger: 'axis',
        axisPointer : {            // 坐标轴指示器，坐标轴触发有效
            type : 'shadow'        // 默认为直线，可选为：'line' | 'shadow'
        }
    },
    legend: {
        data:['支出', '收入']
    },
    grid: {
        left: '3%',
        right: '4%',
        bottom: '3%',
        containLabel: true
    },
    xAxis : [
        {
            type : 'value'
        }
    ],
    yAxis : [
        {
            type : 'category',
            axisTick : {show: false},
            data : ['2002','2003','2004','2005','2006','2007','2008','2009','2010','2011','2012','2013','2014','2015','2016']
        }
    ],
    series : [
        {
            name:'收入',
            type:'bar',
            stack: '总量',
            label: {
                normal: {
                    show: true
                }
            },
            data:[27.4, 37.6, 23.4, 70.5, 90.3, 147.1, 196.5,279.8,344.6,296,343.9,505.3,279.4,312.7,338.4]
        },
        {
            name:'支出',
            type:'bar',
            stack: '总量',
            label: {
                normal: {
                    show: true,
                    position: 'left'
                }
            },
            data:[-15.9, -21.5, -21.8, -33.7, -36.8, -52.5,-69.9,-80.2,-97.7,-103.7,-137.5,-159.4,-168.2,-212.8,-202.8]
        }
    ]
};
```
通过柱状图可以看到有几个明显的变化点，财政收入在2013年暴增和财政收入增减浮动,另外财政支出明显小于财政收入。
![鄞州区财政收支变化柱形图](/img/鄞州区财政收入变化柱形图.jpg)
