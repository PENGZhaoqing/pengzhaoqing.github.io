---
date: 2017-09-04
title: 数据挖掘：如何通过百度地图API抓取物体周边地理位置信息
categories: 学习笔记
tags: [python]
---

## 1.需求描述

对于数据挖掘工程师来说，有时候需要抓取地理位置信息，比如统计房子周边基础设施信息，比如医院、公交车站、写字楼、地铁站、商场等，一般的爬虫可以采用python脚本爬取,有很多成型的框架如scrapy，但是想要爬百度地图就必须遵循它的[JavaScript Api](http://lbsyun.baidu.com/index.php?title=jspopular)，那么肯定需要自己写JavaScript脚本与百度API进行交互，问题是：这种交互下来的数据如何储存（直接写进文本or使用sql数据库？），如何自动化这种交互方式。

因此，本文的目标是用一个rails应用配合js脚本来实现这种自动化抓取和储存，思路是js脚本负责与百度地图Api交互，rails服务器端负责储存抓取的数据，js和rails服务器用ajax方式传递数据. 前提是rails服务器里已经有相应的房屋数据,如房屋的街道地址,小区名字等. 接下来需要做的就是为周边信息数据建表以及相应的关联表(因为它们为多对多关系)

![这里写图片描述](https://github.com/PENGZhaoqing/HousePricing/raw/master/lib/framework_instruction.png)

## 2.流程详解

js代码在用户浏览器中执行，因此爬取的主要部分逻辑都需要写在js脚本里，而rails服务器端需要完成的是获得当前需要抓取的房屋数据以及储存js抓取的数据。下图为对id=1的房屋周边数据抓取的分解过程：

![这里写图片描述](http://img.blog.csdn.net/20170903090817210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


1. 首先由用户在浏览器中点击开始按钮,激活GetDataFromServer()方法,浏览器向rails服务器发送请求,服务器的return_next()方法返回当前需要抓取的房屋数据(主要是街道或者小区的位置信息)
2. 通过getPoint方法,浏览器向Baidu API 发送请求查找房屋坐标,若有结果则继续,否则直接递归调用GetDataFromServer()
3. 使用查询到的房屋坐标搜索周边的信息:对于每一类信息(如地铁,医院等),在查询到结果后立即向服务器发送查询结果以及房屋信息,并标记当前的数据类型(地铁,医院..).服务器在接收到数据后,先判断数据类型,然后根据类别再对房屋的周边信息进行储存.
4. 如果完成当前房屋所有的周边数据的查询后, 再次调用GetDataFromServer()来获得下一个房屋的数据

## 3. 代码实现

### 3.1 浏览器端(js)

1.`GetDataFromServer`: ajax向get_data_url地址以get方法请求json格式的数据, 成功拿到数据后先用小区来匹配房屋坐标, 如果失败再用街道匹配,若两者都没找到结果,那么此房屋的地理信息为空,则查询下一个房屋;若能找到房屋坐标,调用SearchStart()开始搜索周边数据
```
 function GetDataFromServer() {
     $.ajax({
         type: "GET",
         url: get_data_url,
         dataType: 'json',
         success: function (house_data) {
             // 拿到房屋数据后先显示出来
             displayHouseData(house_data);
             // 然后先用街道去查坐标
             myGeo.getPoint(house_data.street, function (point) {
                 if (point) {
                     // 如果查到坐标，开始检索周围信息
                     SearchStart(point, house_data);
                 } else {
                     // 如果街道没查搭配，再用小区去查坐标
                     myGeo.getPoint(house_data.community, function (repoint) {
                         if (repoint) {
                             // 如果查到坐标，开始检索周围信息
                             SearchStart(repoint, house_data)
                         } else {
                             setTimeout(function () {
                                 console.log("Error: no address of " + " id: " + data.id + " community: " + data.community + " street: " + data.street);
                                 // 如果还没查到坐标，继续查询下一个房屋，延迟timeInterval秒
                                 GetDataFromServer();
                             }, timeInterval);
                         }
                     }, "北京市");
                 }
             }, "北京市");
         },
         error: function () {
             alert('error')
         },
         timeout: function () {
             alert('time out')
         }
     });
 } 
```

2.`SearchStart`和`SearchNearby`: SearchStart为SearchNearby的入口, SearchNearby方法构建了一个BMap.LocalSearch对象的函数变量,调用searchNearby并传入关键词就可以查找house_loc附近的所有的包含关键词的位置信息, search_range能指定查找附近的范围. BMap.LocalSearch通过onSearchComplete指定了查询完成后的回调函数:这里我们对查询的结果做一个遍历,计算出这个查询结果与房屋的距离,然后将这些信息整合到一个数组里,传给sendData()来发送数据

```
function SearchStart(point, house_data) {
    // 先在地图上标记出来
    map.centerAndZoom(point, 16);
    map.addOverlay(new BMap.Marker(point));
    // 首先查询此房屋的第一个关键词信息（公交车站，idx＝0）
    setTimeout(function () {
        SearchNearby(point, house_data, 0);
    }, timeInterval);
}
function SearchNearby(house_loc, house_data, keyword_idx) {
    var nearby_info = [];
    // 清除地图覆盖物
    map.clearOverlays();
    var local = new BMap.LocalSearch(map, {
        renderOptions: {map: map, autoViewport: false},
        pageCapacity: 50,
        onSearchComplete: function (results) {
            DisplayClear();
            if (local.getStatus() == BMAP_STATUS_SUCCESS) {
                // 百度地图成功返回，将每个周边信息储存到nearby_info里
                for (var i = 0; i < results.getCurrentNumPois(); i++) {
                    var locate = results.getPoi(i);
                    if (locate != null) {
                        // 查询结果与房屋的距离
                        var distance = parseFloat(map.getDistance(locate.point, house_loc)).toFixed(1);
                        nearby_info.push(locate.title + "/" + locate.point.lng + '/' + locate.point.lat + '/' + distance);
                        DisplayNearbyData(nearby_info, locate, distance)
                    }
                }
                // 获得百度地图查询结果后立即发送给服务器
                return sendData(keywords_en[keyword_idx], nearby_info, house_data, house_loc, keyword_idx)
            } else {
                GetDataFromServer();
                console.log("No records with baiduAPI:", local.getStatus());
                return false;
            }
        }
    });
    local.searchNearby(keywords[keyword_idx], house_loc, search_range);
}
```

3.`sendData`: sendData负责发送查询数据nearby_info, 周边数据类型由nearby_type指定,房子本身的数据信息由house_data提供而坐标由house_loc给出, idx记录着现在查询的关键词的索引. sendData使用ajax post方法提交数据, 当提交成功后, 通过调用SearchNearby并传递下一个关键词的id来检索这个房子其他周边信息;如果当前关键词已经是最后一个,那么调用GetDataFromServer来启动下一轮的查询
```
function sendData(nearby_type, nearby_info, house_data, house_loc, idx) {
    data = "nearby_type=" + nearby_type + "&nearby_info=" + nearby_info + "&id=" + house_data.id + "&lat=" + house_loc.lat + "&lng=" + house_loc.lng;
    $.ajax({
        type: "POST",
        url: post_data_url,
        data: data,
        dataType: "JSON",
        success: function (data) {
            if (flag) {
                console.log("warning", 'pause');
            } else {
                // 当查询到最后一个kewords时，请求服务器获得下一个房屋信息
                if (idx == keywords.length - 1) {
                    GetDataFromServer();
                } else {
                    // 查询此房屋的下一个关键词信息
                    setTimeout(function () {
                        SearchNearby(house_loc, house_data, idx + 1);
                    }, timeInterval);
                }
                console.log("success", data);
            }
            return true;
        },
        error: function () {
            alert('error in post');
            return false;
        },
        timeout: function () {
            alert('time out in post');
            return false;
        }
    });
}
```

更多详细的信息请见[index.html.erb](https://github.com/PENGZhaoqing/HousePricing/blob/master/app/views/spiders/index.html.erb)中的js代码

### 3.2 服务器端(rails controller)

#### SpidersController

1.`return_next`:  通过类变量@@house_id确定当前需要查询的房屋id,这个全局id变量随着return_text的调用而自增. 为了避免重复抓取, 跳过已经有相关记录的,最后以json格式返回房屋数据

```
@@house_id=0

def return_next
  # 查询下一个房屋信息
  house=House.next_record(@@house_id)
  @@house_id=house.id

  # 避免重复抓取，跳过已经有相关信息的
  while not house.buses_houses.nil? and not house.buses_houses.blank?
    house=House.next_record(@@house_id)
    @@house_id=house.id
    break if house == House.last
  end

  # TODO
  # 避免重复抓取，现在只能靠bus信息进行判断，希望更全面的信息判断
  # END

  if house == House.last
    redirect_to buses_path, flash: {:success => "抓取完毕"}
  else
    respond_to do |format|
      format.json { render :json => house }
    end
  end
end
```

2.`create`: 接受抓取的周边数据,判断数据类型并交给insert处理

```
def create
  house=House.find_by(id: params[:id])
  house.latitude=params[:lat]
  house.longitude=params[:lng]
  house.save

  insert(house, params, Bus, BusesHouses, 'bus') if params[:nearby_type] == 'bus'
  insert(house, params, Hospital, HospitalsHouses, 'hospital') if params[:nearby_type] == 'hospital'
  insert(house, params, Work, WorksHouses, 'work') if params[:nearby_type] == 'work'
  insert(house, params, School, SchoolsHouses, 'school') if params[:nearby_type] == 'school'
  insert(house, params, Subway, SubwaysHouses, 'subway') if params[:nearby_type] == 'subway'
  insert(house, params, Shop, ShopsHouses, 'shop') if params[:nearby_type] == 'shop'

  render json: params.as_json
end
```

#### HousesHelper

3.`insert`: 解析sendData()发送来的数据, 创建相应的记录.  attr中存放着每条周边数据经度,维度以及名字; obj为类名,如Bus, Subway, 通过find_by()方法查询这个经度和维度是否已经存在,若已经存在此记录, 说明之前存过了,因为同一片区域的房子可能会有公有的基础设施; 若不存在,则创建新的记录. asso_obj为关联表,如BusesHouses, 这是由于bus和house为多对多关系: 一个公交车站附近有多个房屋,一个房屋附近也有多个公交车站, 所以需要这个关联表来储存bus和house的对应关系(由三个字段表示: house_id, bus_id和distance, 表示这个house_id与这个bus_id是附近关系,而且相距distance), 因此有几类周边类型,就需要多少个关联表,本文共有Bus, Hospital, Work, School, Subway, Shop 6类周边数据, 所以需要维护6个关联表,BusHouses是其中的一种.

```
def insert(house, params, obj, asso_obj, asso_type)
  params[:nearby_info].split(',').each do |row|
    attr=row.split('/')
    bus=obj.find_by(longitude: attr[1], latitude: attr[2])
    if bus.nil?
      bus=obj.new(name: attr[0], longitude: attr[1], latitude: attr[2])
      bus.save
    end
    asso_obj.create("#{asso_type}_id": bus.id, house_id: house.id, distance: attr[3])
  end
end
```

## Demo

https://house-pricing.herokuapp.com/

先给出一些截图, 目前基础数据只爬取了部分北京二手房的数据,所以数据可能已经过时了, 完整的代码在[HousePricing](https://github.com/PENGZhaoqing/HousePricing), 房屋的基础数据可以去房天下,链家等房价网站爬取, 参考这个[scrapy-HousePricing](https://github.com/PENGZhaoqing/scrapy-HousePricing), 爬取后导入到rails的数据库,就可以使用上面的方法便利抓取地理位置信息

![这里写图片描述](https://github.com/PENGZhaoqing/HousePricing/raw/master/lib/screen1.png)

![这里写图片描述](https://github.com/PENGZhaoqing/HousePricing/raw/master/lib/screen5.png)
