<script setup>

import { onMounted, ref } from 'vue';
import mapboxgl from 'mapbox-gl'
import MapboxLanguage from '@mapbox/mapbox-gl-language'


const position = ref([])
const carTrace = ref([])
const oneCarTrace = ref([])
const pointColor = ref('#4264fb')
const clickedId = ref()
let clickFlag = false
// 用于控制是否正在更新
const isUpdating = ref(false)
let currentIndex = ref(0); // 当前的数据索引
let intervalId = null; // 用于存储定时器的 ID
const updateInterval = 1000; // 更新间隔，单位毫秒
let map
const updatePosition = (data) => {
    position.value = []
    data.forEach(element => {
        position.value.push({
            'type': 'Feature',
            'properties': {
                'id': element.slice(0, 1)
            },
            'geometry': {
                'type': 'Point',
                'coordinates': element.slice(1, 3)
            }
        })
    })
}
// 创建地图
const createMap = () => {
    mapboxgl.accessToken = 'pk.eyJ1IjoiZnJhbmRlcm1hbm4iLCJhIjoiY2xramlxbW5xMHB6OTN0cWx2YXdyOWMyNiJ9.11h9QepknHdlVU2j-0r5EA';
    map = new mapboxgl.Map({
        container: 'map', // container id 绑定的组件的id
        style: 'mapbox://styles/mapbox/streets-v11', //地图样式，可以使用官网预定义的样式,也可以自定义
        center: [114.07750325, 22.5490834767], // 初始地图中心点位置
        zoom: 11,     // starting zoom 地图初始的层级
        antialias: true, //抗锯齿，通过false关闭提升性能
    });
    map.addControl(new MapboxLanguage({ defaultLanguage: 'zh-Hans' }))//切换成中文

}
const removeLayerAndSource = id => {
    if (map.getLayer(id)) {
        map.removeLayer(id);
    }
    if (map.getSource(id)) {
        map.removeSource(id);
    }
}

// 更新点
const updateMap = () => {
    removeLayerAndSource('places')
    map.addSource('places', {
        'type': 'geojson',
        'data': {
            'type': 'FeatureCollection',
            'features': position.value
        }
    })
    map.addLayer({
        'id': 'places',
        'type': 'circle',
        'source': 'places',
        'paint': {
            'circle-color': pointColor.value,
            'circle-radius': 5,
            'circle-stroke-width': 1,
            'circle-stroke-color': '#fff'
        }
    })
}

// 画轨迹
const drawTrace = () => {
    removeLayerAndSource('oneCar')
    removeLayerAndSource('route')
    carTrace.value = []
    oneCarTrace.value = []
    let index = 1
    const promises = []
    while (index <= currentIndex.value) {
        const promise = import(`../../store/vehicle/vehicle${index}.json`).then(data => {
            data.default.forEach(element => {
                if (element[0] == clickedId.value) {
                    carTrace.value.push(element.slice(1, 3))
                    oneCarTrace.value.push({
                        'type': 'Feature',
                        'properties': {
                            'id': element.slice(0, 1)
                        },
                        'geometry': {
                            'type': 'Point',
                            'coordinates': element.slice(1, 3)
                        }
                    })
                }
            })

        })
        promises.push(promise)
        index = index + 1
    }
    Promise.all(promises).then(() => {
        map.addSource('oneCar', {
            'type': 'geojson',
            'data': {
                'type': 'FeatureCollection',
                'features': oneCarTrace.value
            }
        })
        map.addLayer({
            'id': 'oneCar',
            'type': 'circle',
            'source': 'oneCar',
            'paint': {
                'circle-color': '#4264fb',
                'circle-radius': 5,
                'circle-stroke-width': 1,
                'circle-stroke-color': '#fff'
            }
        })
        map.addSource('route', {
            'type': 'geojson',
            'data': {
                'type': 'Feature',
                'properties': {},
                'geometry': {
                    'type': 'LineString',
                    'coordinates': carTrace.value
                }
            }
        })
        map.addLayer({
            'id': 'route',
            'type': 'line',
            'source': 'route',
            'layout': {
                'line-join': 'round',
                'line-cap': 'round'
            },
            'paint': {
                'line-color': '#4264fb',
                'line-width': 4
            }
        })
    })
}

// 更新数据
const startUpdating = () => {
    if (!intervalId) {
        intervalId = setInterval(() => {
            // 更新当前数据索引，确保不超过数据的长度
            currentIndex.value = (currentIndex.value + 1)
            // 按照当前索引加载数据并更新地图
            import(`../../store/vehicle/vehicle${currentIndex.value}.json`).then(data => {
                updatePosition(data.default)
                if (clickFlag) {
                    pointColor.value = ('#778899')
                    updateMap()
                    drawTrace(clickedId.value)
                } else {
                    pointColor.value = ('#4264fb')
                    updateMap()
                }
            })
            if (currentIndex.value >= 50) {
                clearInterval(intervalId);
                intervalId = null;
            }
        }, updateInterval)

    }
}
// 暂停
const pauseUpdating = () => {
    if (intervalId) {
        clearInterval(intervalId);
        intervalId = null;
    }
}
// 重新开始
const restart = () => {
    currentIndex.value = 0
    pointColor.value = ('#4264fb')
    removeLayerAndSource('places')
    removeLayerAndSource('route')
    removeLayerAndSource('oneCar')
}


onMounted(() => {
    createMap()
    map.on('mouseenter', 'places', (e) => {
        // 更改光标形状为手指
        map.getCanvas().style.cursor = 'pointer'
    })
    map.on('mouseleave', 'places', () => {
        // 恢复默认的光标形状
        map.getCanvas().style.cursor = ''
    });
    map.on('click', 'places', (e) => {
        const clickedFeature = e.features[0]// 获取点击的特征
        clickedId.value = clickedFeature.properties.id.slice(1, -1) //车辆id
        console.log('id', clickedId.value)
        console.log('裁剪id', clickedId.value)
        clickFlag = !clickFlag
        if (clickFlag) {
            pointColor.value = ('#778899')
            updateMap()
            drawTrace(clickedId.value)
        } else {
            removeLayerAndSource('route')
            removeLayerAndSource('oneCar')
            pointColor.value = ('#4264fb')
            updateMap()
        }

    })
})
</script>
<template>
    <div id="map"></div>
    <button @click="startUpdating">开始</button>
    <button @click="pauseUpdating">暂停</button>
    <button @click="restart">重新开始</button>
    <p>{{ currentIndex }}/300</p>
</template>

<style scoped>
#map {
    width: 80vw;
    height: 80vh;
}
</style>