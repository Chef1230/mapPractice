安装mapbox-gl的依赖

```shell
yarn add mapbox-gl
```

引入包

```js
import mapboxgl from 'mapbox-gl'
```

创建容器

```html
<div id="map"></div>
```

初始化

```js
const createrMap = () => {
    mapboxgl.accessToken = 'pk.eyJ1IjoiZnJhbmRlcm1hbm4iLCJhIjoiY2xramlxbW5xMHB6OTN0cWx2YXdyOWMyNiJ9.11h9QepknHdlVU2j-0r5EA';
    let map = new mapboxgl.Map({
        container: 'map', // container id 绑定的组件的id
        style: 'mapbox://styles/mapbox/streets-v11', //地图样式，可以使用官网预定义的样式,也可以自定义
        center: [113.27, 23.13], // 初始地图中心点位置
        zoom: 15,     // starting zoom 地图初始的层级
        antialias: true, //抗锯齿，通过false关闭提升性能
    });
    map.addControl(new MapboxLanguage({ defaultLanguage: 'zh-Hans' }))
}
```

页面渲染完成后执行

```js
onMounted(
    createrMap
)
```

