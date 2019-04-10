---
title: ECharts 奇技淫巧
date: 2019-04-04 17:44:40
categories:
- frontend
tags:
- frontend
- echarts
---
## 多系列阶梯瀑布图

![多系列阶梯瀑布图](/images/多系列阶梯瀑布图.png)

```javascript
// utilities
const fromEntries = entries =>
  Object.assign(...entries.map(([ key, value ]) => ({ [key]: value })))

// patterns
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');

const createNegativePattern = color => {
  return new Promise((resolve, reject) => {
    const fill = encodeURIComponent(color);
    const img = new Image(6, 6);
    img.src = `data:image/svg+xml,%3Csvg width='6' height='6' viewBox='0 0 6 6' xmlns='http://www.w3.org/2000/svg'%3E%3Cg fill='${fill}' fill-opacity="1" fill-rule='evenodd'%3E%3Cpath d='M1 0H0 L6 6V5zM0 5v1H1z'/%3E%3C/g%3E%3C/svg%3E`;
    img.onload = e => {
      const pattern = ctx.createPattern(img, 'repeat');
      resolve(pattern);
    };
  });
};

const negativePatterns = new Map();
['#fc4f3f', '#202126'].forEach(color => {
  createNegativePattern(color)
    .then(pattern => {
      negativePatterns.set(color, pattern);
      myChart.setOption(option);
    });
});

// ratios of category width
const BAR_GAP = 0.1;
const CAT_GAP = 0.3; // gap between categories

function generateItemRenderer(index, count, startsWith) {
  return function(params, api) {
    const x = api.value(0);
    const fromY = params.context.lastValue || startsWith;
    const toY = api.value(1);
    const start = api.coord([x, toY]);
    const end = api.coord([x, fromY]);
    const toSize = api.size([x, toY]);
    const style = api.style();

    if (fromY > toY) {
      style.stroke = style.fill;
      style.fill = negativePatterns.get(style.fill) || '#fff';
    }

    params.context.lastValue = toY;

    const barWidth = toSize[0] * ((1 - CAT_GAP) - BAR_GAP * (count - 1)) / count;
    const barOffset = toSize[0] * (CAT_GAP / 2 + BAR_GAP * index) + barWidth * index;
    const barShape = fromEntries(Object.entries({
      x: Math.round(start[0] - (toSize[0] / 2) + barOffset),
      y: Math.round(Math.min(start[1], end[1])),
      width: Math.round(barWidth),
      height: Math.round(Math.abs(end[1] - start[1]))
    })
      .map(([k, v]) => [k, Math.round(v)])
      .map(([k, v]) => [k, ['x', 'y'].includes(k) ? v + 0.5 : v]))

    console.log(barShape);
    const bar = {
      type: 'rect',
      shape: barShape,
      style
    };
    const label = {
      type: 'text',
      style: {
        text: toY.toFixed(0),
        textAlign: 'center',
        x: barShape.x + barWidth * 0.5,
        y: barShape.y + (fromY < toY ? -20 : barShape.height + 8),
        fill: style.stroke || style.fill
      }
    };

    return {
      type: 'group',
      children: [bar, label]
    };
  };
}

option = {
  xAxis: {
    name: '时间（s）',
    type: 'category',
    data: new Array(8).fill(0).map((val, idx) => idx + 1)
  },
  yAxis: [{
    name: '扭矩（N·m）',
    min: -100,
    max: 100,
    interval: 20,
    type: 'value'
  }, {
    show: false
  }],
  series: [{
    name: '扭矩',
    type: 'custom',
    renderItem: generateItemRenderer(0, 2, 0),
    itemStyle: {
      normal: { color: '#fc4f3f' }
    },
    data: [-27, 19, 9, 20, 11, 13]
   }, {
    name: '扭矩',
    type: 'custom',
    renderItem: generateItemRenderer(1, 2, 0),
    itemStyle: {
      normal: { color: '#202126' }
    },
    data: [-33, 18, 6, 20, 8, 11]
  }],
  animation: false,
}
```

## 雷达图指示器处显示数据值

![雷达图指示器处显示数据值](/images/雷达图指示器处显示数据值.png)

```javascript
const radarIndicators = [
  '总做功\n{v|（%% J）}', '峰值扭矩\n最大值\n{v|（%% Nm）}',
  '峰值扭矩\n均值\n{v|（%% Nm）}', '疲劳指数\n{v|（%%）}',
  '做功均值\n{v|（%% J）}', '功率均值\n{v|（%% W）}'
];
const radarMaxima = [1000, 100, 100, 1, 100, 15];

function getRadarFormatter(data) {
  const radarData = data;

  return function(text, indicator) {
    const idx = parseInt(text);
    return radarIndicators[idx].replace('%%', radarData[idx].toFixed(1));
  };
}

option = {
  title: {
    text: '屈曲参数',
    textStyle: {
      color: '#202126',
      fontSize: 14,
      fontWeight: 'normal'
    },
    padding: [16, 12]
  },
  radar: {
    name: {
      formatter: getRadarFormatter([0, 0, 0, 0, 0, 0]),
      color: '#626262',
      lineHeight: 16,
      rich: {
        v: {
          color: '#202126',
          fontWeight: 'bold',
          lineHeight: 24
        }
      }
    },
    nameGap: 10,
    indicator: radarMaxima.map((max, idx) => ({ text: String(idx), max })),
    center: ['50%', '55%'],
    radius: '45%'
  },
  series: {
    name: '屈曲参数',
    tooltip: { trigger: 'item' },
    type: 'radar',
    color: ['#fc4f3f'],
    lineStyle: {
      normal: {
        width: 1
      }
    },
    areaStyle: {
      normal: {
        opacity: 0.42
      }
    },
    data: [{ value: [0, 0, 0, 0, 0] }]
  },
  animation: false
};

function updateRadarGraph(radarData) {
  myChart.setOption({
    radar: {
      name: {
        formatter: getRadarFormatter(radarData)
      },
      indicator: radarMaxima
        .map((max, idx) => ({ text: String(idx), max }))
    },
    series: [{
      type: 'radar',
      data: [{ value: radarData }]
    }]
  });
}

setTimeout(function() {
  updateRadarGraph([233, 66, 45, 0.2, 23.3, 4.5])
}, 0);

```
