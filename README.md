# 从API获取地理坐标
由于爬虫经常会因为网站的策略变化而失效，所以这是一项时效性非常强的技术，中文互联网上大多关于爬虫的教学、链接都已过时，因此本人会尽可能在最新的环境下进行说明

**爬取平台：使用高德地图开放平台**

首先进入[行政区查询](https://lbs.amap.com/api/javascript-api/guide/services/district-search)

在本页面中的**获取下级行政区信息**中的链接：https://lbs.amap.com/demo/javascript-api/example/district-search/city-drop-down-list中可以尝试选择地区进行查询

关于行政区域查询API服务的详细信息，查看以下网址：

https://lbs.amap.com/api/webservice/guide/api/district/

总之，爬取API数据的原理就是根据自己的需要设置参数，然后将参数拼接到url中访问，因此要先搞明白我们爬取的`url`到底是什么

## url拼接

API接口由该url提供：

```python
url = 'https://restapi.amap.com/v3/config/district?'
```

之后接上我们的`parameters`

```python
keywords = '津南区'
my_key = '这里要替换成你的key'
'''
subdistrict取值：
0：不返回下级行政区；

1：返回下一级行政区；

2：返回下两级行政区；

3：返回下三级行政区；
'''
dict = {
    'key' : my_key,
    'keywords' : keywords,
    'subdistrict' : '1',
    'extensions' : 'all'
}

url_data = parse.urlencode(dict) #将字典拼起来，作为参数
address = url + url_data #将参数传到整个url中，即最终要用来访问的url
```

## 访问url抓取数据

接下来用得到的`address`进行request请求

```python
def get_html(url):
    # 创建访问器
    request = urllib.request.Request(url)
    # 访问网页
    response = urllib.request.urlopen(request)
    # 读取网页内容
    webpage = response.read()
    return webpage

webpage = get_html(address)
```

注意这里要用到的库为

```python
from urllib import request
```

而不是经常使用的`requests`，在这里对二者进行一下区别

### urllib库

urllib库中的response对象是先创建http.resquest对象，装载到`request.urlopen`中完成http请求

返回的是`http.response`对象，实际上是html属性，使用`response.read()`进行读取

随后还需要使用`decode()`进行解码后，中文字符才能够显示出来

在本次爬取中，就使用了json解析：

```python
data = json.loads(webpage.decode('utf-8', 'ignore'))
```

`ignore`参数表示不将中文字符转换为unicode编码

### requests库

requests库调用`request.get`方法传入url和参数，返回的对象是response对象，打印出来的是响应状态码

通过`.text`方法可以返回unicode型的数据，在网页的header中定义的编码形式，而使用`.content`可以返回bytes二进制的数据，`.json`则可以返回json

### 区别

一般情况更推荐使用requests库，因为requests可以直接构造`get`, `post`请求并发起，而urllib.request只能先构造`get`, `post`请求再发起，因此requests库更为便捷

## 绘制与保存

### 绘制

解码后，我们就得到了我们想要的数据，它目前是以`json`的格式存在的，在本次爬取的数据中，我们要获得津南区的轮廓图，对于返回的数据，只需要取得`districts`栏中的第一项即可，因为后面的属于下一级行政区了，所以我们只需要提取其中的`polyline`

```python
polyline = data['districts'][0]['polyline']
polyline = polyline.split(';') #对每一项进行分割
p = []
for i in polyline:
    a, b = i.split(',') #获取一个点的经纬度
    p.append([float(a), float(b)])
```

转换为浮点数后就可以引入`geopandas`保存数据了

```python
import geopandas
from shapely.geometry import Polygon

geodata = pd.DataFrame([data['districts'][0]])
geodata['geometry'] = [Polygon(p)]
```

这里的`geodata`数据类型为`pandas.core.frame.DataFrame`

我们要使用`geopandas`绘制的话还需要进一步转换

```python
geodata = geopandas.GeoDataFrame(geodata)
```

转换后的数据类型为`geopandas.geodataframe.GeoDataFrame`

接下来

```python
geodata.polt()
```

进行绘制

<img src="https://mdstore.oss-cn-beijing.aliyuncs.com/Screenshot%202023-11-09%20at%2015.08.46.png" alt="Screenshot 2023-11-09 at 15.08.46" style="zoom: 33%;" />

### 保存数据

```python
geodata = geodata.drop(['polyline', 'districts'], axis = 1)
geodata.to_file(r'津南区')

#如果要保存为GeoJSON格式
geodata.to_file(r'津南区.json', driver = 'GeoJSON')
```

可以在https://geojson.io上打开`.json`数据

<img src="https://mdstore.oss-cn-beijing.aliyuncs.com/Screenshot%202023-11-09%20at%2015.10.16.png" alt="Screenshot 2023-11-09 at 15.10.16" style="zoom: 25%;" />

我们发现，高德地图上的经纬度信息并不能与该网站上的信息吻合，原因在于二者使用的坐标系不同，需要进行坐标转换，坐标转换的内容由于篇幅关系将写在另一篇文章中
