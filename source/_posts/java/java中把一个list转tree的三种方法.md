---
title: java中把一个list转tree的三种方法
abbrlink: bc773bb5
date: 2020-03-21 13:00:32
tags:
---
说到java把list转tree，网上有一大堆文章，但是我看过之后发现基本都只说了递归和两层嵌套循环两种方法，没人提到两次遍历的方法，我今天就把三种方法都实现以下，做一下对比
<!-- more-->
# 问题
周二面试中，面试官提了一个问题，当时答得不是特别好，手写代码能力还是不行啊，一个是比较紧张，一个是代码没法调试，写递归的时候给自己绕晕了。下面是问题：
> 我们有个需求，数据库要存一个无限级联的tree，比如菜单，或者地区等数据，现有两个问题，1.问如何设计表，2.怎么返回给前端一个无线级联的json数据；
# 思考
第一个问题，设计表的话，拿地区举例子,这个只要有id，parentId，就没啥问题。主要是第二个问题
``` sql
create table Zone {
    id varchar(255)
    name varchar(255)
    code varchar(255)
    parentId varchar(255)
}
```

第二个问题，我的想法是通过一个sql查询查出来所有数据，得到一个 Zone集合，然后就回到了主题，如何用java把list转tree。我第一想法是递归。递归的话，需要考虑几个因素，1.终止条件；2.处理逻辑，3.参数（数据参数，当前层级），4.返回值，然后套入这个问题，分析如下：
1. 中断条件：当前节点，无子节点，终止退出
2. 处理逻辑：根据parentId查找节点，设置到parent的children属性中
3. 参数：数据就是list集合，当前层级参数是parent节点
4. 无

---
# 准备工作
首先我们需要一些辅助类，代码如下（避免代码冗长，我就不写get，set方法了，理解意思就行，实际项目中肯定不能写么写）

地区类：Zone.class
``` java

import java.util.List;

public class Zone {
    String id;
    String name;
    String parentId;
    List<Test.Zone> children;

    public Zone(String id, String name, String parentId) {
        this.id = id;
        this.name = name;
        this.parentId = parentId;
    }
    public void addChildren(Zone zone){
        if(children == null) {
            children = new ArrayList<>();
        }
        children.add(zone);
    }
}

```
测试类：Test.java
``` java
import java.util.ArrayList;
import java.util.List;

public class Test {

    public static void main(String[] args) {
        List<Zone> zoneList = new ArrayList<>();
        zoneList.add(new Zone("1","上海","0"));
        zoneList.add(new Zone("2","北京","0"));
        zoneList.add(new Zone("3","河南","0"));
        zoneList.add(new Zone("31","郑州","3"));
        zoneList.add(new Zone("32","洛阳","3"));
        zoneList.add(new Zone("321","洛龙","32"));
        zoneList.add(new Zone("11","松江","1"));
        zoneList.add(new Zone("111","泗泾","11"));
        List<Zone> rootZone1 = ZoneUtils.buildTree1(zoneList); // 测试第一种方法
        List<Zone> rootZone2 = ZoneUtils.buildTree2(zoneList); // 测试第二种方法
        List<Zone> rootZone3 = ZoneUtils.buildTree3(zoneList); // 测试第三种方法
    }
}
```
地区工具类，提供三种构建tree的方法：ZoneUtils.java
``` java

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class ZoneUtils {
    public static List<Zone> buildTree1(List<Zone> zoneList) {
        // TODO : 第一种解法
        return null;
    }
    public static List<Zone> buildTree2(List<Zone> zoneList) {
        // TODO : 第二种解法
        return null;
    }
    public static List<Zone> buildTree3(List<Zone> zoneList) {
        // TODO : 第三种解法
        return null;
    }
}

```
# 第一种方法：递归

``` java
public static List<Zone> buildTree1(List<Zone> zoneList) {
    List<Zone> result = new ArrayList<>();
    for (Zone zone:zoneList) {
        if (zone.parentId.equals("0")) {
            result.add(zone);
            setChildren(zoneList, zone);
        }
    }
    return result;
}

public static void setChildren(List<Zone> list, Zone parent) {
    for (Zone zone: list) {
        if(parent.id.equals(zone.parentId)){
            parent.children.add(zone);
        }
    }
    if (parent.children.isEmpty()) {
        return;
    }
    for (Zone zone: parent.children) {
        setChildren(list, zone);
    }
}

```
# 第二种方法：两层循环
``` java
public static List<Zone> buildTree2(List<Zone> zoneList) {
    List<Zone> result = new ArrayList<>();
    for (Zone zone : zoneList) {
        if (zone.parentId.equals("0")) {
            result.add(zone);
        }
        for (Zone child : zoneList) {
            if (child.parentId.equals(zone.id)) {
                zone.addChildren(child);
            }
        }
    }
    return result;
}
```
# 第三种方法：两次遍历
``` java
public static List<Zone> buildTree3(List<Zone> zoneList) {
    Map<String, List<Zone>> zoneByParentIdMap = new HashMap<>();
    Map<String, Zone> zoneByIdMap = new HashMap<>();
    zoneList.forEach(zone -> {
        zoneByIdMap.put(zone.id, zone);
        List<Zone> children = zoneByParentIdMap.getOrDefault(zone.parentId, new ArrayList<Zone>());
        children.add(zone);
        zoneByParentIdMap.put(zone.parentId, children);
    });
    zoneByParentIdMap.forEach((key, value) -> {
        Zone zone = zoneByIdMap.get(key);
        if (zone != null) zone.children = value;
    });
    return zoneByIdMap.values().stream()
            .filter(v -> v.parentId.equals("0"))
            .collect(Collectors.toList());
}
```
# 三种方法对比
前两种方法的时间复杂度都和叶子节点的个数相关，我们假设叶子节点个数为m
**方法一:** 用递归的方法，时间复杂度等于：O(n +（n-m）* n)，根据**初始算法**那篇文章的计算时间复杂度的方法，可以得到最终时间复杂度是O(n<sup>2</sup>)
**方法二:** 用两层嵌套循环的方法，时间复杂度等于：O(n +（n-m）* n)，和方法一的时间复杂度是一样的，最终时间复杂度是O(n<sup>2</sup>)
**方法三:** 用两次遍历的方法，时间复杂度等于：O(3n)，根据**初始算法**那篇文章的计算时间复杂度的方法，可以得到最终时间复杂度是O(n)，但它的空间复杂度比前两种方法稍微大了一点，但是也是线性阶的，所以影响不是特别大。所以第三种方法是个人觉得比较优的一种方法

|-|代码执行次数|时间复杂度|代码简洁程度|
|-|-|-|-|
|方法1|O(n +（n-m）* n)|指性阶,O(n<sup>2</sup>) |一般|
|方法2|O(n +（n-m）* n)|指性阶,O(n<sup>2</sup>) |良好|
|方法3|O(3n)|线性阶,O(n|复杂|