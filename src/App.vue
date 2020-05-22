<template>
  <div class="container-fluid">
    <div id="map" class="map"></div>
   
  </div>
</template>

<script>
import * as ol from 'ol'
import 'ol/ol.css';
import Map from 'ol/Map';
import {unByKey} from 'ol/Observable';
import Overlay from 'ol/Overlay';
import {getArea, getLength} from 'ol/sphere';
import View from 'ol/View';
import {LineString, Polygon} from 'ol/geom';
import Draw from 'ol/interaction/Draw';
import {Tile as TileLayer, Vector as VectorLayer} from 'ol/layer';
import {OSM, Vector as VectorSource} from 'ol/source';
import {Circle as CircleStyle, Fill, Stroke, Style} from 'ol/style';
import XYZ from 'ol/source/XYZ';


export default {
  mounted: function() {
    this.init()
  },
  methods: {
    init: function() {
          let raster = new TileLayer({
          source: new OSM()
        });

let source = new VectorSource();

let vector = new VectorLayer({
  source: source,
  style: new Style({
    fill: new Fill({
      color: 'rgba(255, 255, 255, 0.2)'
    }),
    stroke: new Stroke({
      color: '#ffcc33',
      width: 2
    }),
    image: new CircleStyle({
      radius: 7,
      fill: new Fill({
        color: '#ffcc33'
      })
    })
  })
});

let sketch;
let helpTooltipElement;
let helpTooltip;
let measureTooltipElement;
let measureTooltip;
let continuePolygonMsg = '单击以继续绘制多边形';
let continueLineMsg = '点击继续画线';

let pointerMoveHandler = function(evt) {
  if (evt.dragging) {
    return;
  }
  let helpMsg = '点击开始绘图';

  if (sketch) {
    let geom = sketch.getGeometry();
    if (geom instanceof Polygon) {
      helpMsg = continuePolygonMsg;
    } else if (geom instanceof LineString) {
      helpMsg = continueLineMsg;
    }
  }

  helpTooltipElement.innerHTML = helpMsg;
  helpTooltip.setPosition(evt.coordinate);

  helpTooltipElement.classList.remove('hidden');
};

let map = new ol.Map({
    layers: [
    new TileLayer({
      source: new XYZ({
        url: 'http://api.tianditu.gov.cn/staticimage?center=116.78880247486877,33.95520562955379&width=400&height=300&zoom=10&tk=a4e7559608e463ccfceb274142c0392a'
      })
    })
  ],
    target: 'map',
    view: new ol.View({
      center: [116.78880247486877, 33.95520562955379],
      zoom: 2,
    })
  })

map.on('pointermove', pointerMoveHandler);

map.getViewport().addEventListener('mouseout', function() {
  helpTooltipElement.classList.add('hidden');
});

let typeSelect = document.getElementById('type');

let draw; 
let formatLength = function(line) {
  let length = getLength(line);
  let output;
  if (length > 100) {
    output = (Math.round(length / 1000 * 100) / 100) +
        ' ' + 'km';
  } else {
    output = (Math.round(length * 100) / 100) +
        ' ' + 'm';
  }
  return output;
};

let formatArea = function(polygon) {
  let area = getArea(polygon);
  let output;
  if (area > 10000) {
    output = (Math.round(area / 1000000 * 100) / 100) +
        ' ' + 'km<sup>2</sup>';
  } else {
    output = (Math.round(area * 100) / 100) +
        ' ' + 'm<sup>2</sup>';
  }
  return output;
};

function addInteraction() {
  let type = 'LineString';
  draw = new Draw({
    source: source,
    type: type,
    style: new Style({
      fill: new Fill({
        color: 'rgba(255, 255, 255, 0.2)'
      }),
      stroke: new Stroke({
        color: 'rgba(0, 0, 0, 0.5)',
        lineDash: [10, 10],
        width: 2
      }),
      image: new CircleStyle({
        radius: 5,
        stroke: new Stroke({
          color: 'rgba(0, 0, 0, 0.7)'
        }),
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)'
        })
      })
    })
  });
  map.addInteraction(draw);

  createMeasureTooltip();
  createHelpTooltip();

  let listener;
  draw.on('drawstart',
    function(evt) {
      sketch = evt.feature;
      let tooltipCoord = evt.coordinate;

      listener = sketch.getGeometry().on('change', function(evt) {
        let geom = evt.target;
        let output;
        if (geom instanceof Polygon) {
          output = formatArea(geom);
          tooltipCoord = geom.getInteriorPoint().getCoordinates();
        } else if (geom instanceof LineString) {
          output = formatLength(geom);
          tooltipCoord = geom.getLastCoordinate();
        }
        measureTooltipElement.innerHTML = output;
        measureTooltip.setPosition(tooltipCoord);
      });
    });

  draw.on('drawend',
    function() {
      measureTooltipElement.className = 'ol-tooltip ol-tooltip-static';
      measureTooltip.setOffset([0, -7]);
      // unset sketch
      sketch = null;
      measureTooltipElement = null;
      createMeasureTooltip();
      unByKey(listener);
    });
}

function createHelpTooltip() {
  if (helpTooltipElement) {
    helpTooltipElement.parentNode.removeChild(helpTooltipElement);
  }
  helpTooltipElement = document.createElement('div');
  helpTooltipElement.className = 'ol-tooltip hidden';
  helpTooltip = new Overlay({
    element: helpTooltipElement,
    offset: [15, 0],
    positioning: 'center-left'
  });
  map.addOverlay(helpTooltip);
}
function createMeasureTooltip() {
  if (measureTooltipElement) {
    measureTooltipElement.parentNode.removeChild(measureTooltipElement);
  }
  measureTooltipElement = document.createElement('div');
  measureTooltipElement.className = 'ol-tooltip ol-tooltip-measure';
  measureTooltip = new Overlay({
    element: measureTooltipElement,
    offset: [0, -15],
    positioning: 'bottom-center'
  });
  map.addOverlay(measureTooltip);
}

addInteraction();
}
  }
}


</script>

<style >
 .map {
        width: 100%;
        height:400px;
      }
      .ol-tooltip {
        position: relative;
        background: rgba(0, 0, 0, 0.5);
        border-radius: 4px;
        color: white;
        padding: 4px 8px;
        opacity: 0.7;
        white-space: nowrap;
        font-size: 12px;
      }
      .ol-tooltip-measure {
        opacity: 1;
        font-weight: bold;
      }
      .ol-tooltip-static {
        background-color: #ffcc33;
        color: black;
        border: 1px solid white;
      }
      .ol-tooltip-measure:before,
      .ol-tooltip-static:before {
        border-top: 6px solid rgba(0, 0, 0, 0.5);
        border-right: 6px solid transparent;
        border-left: 6px solid transparent;
        content: "";
        position: absolute;
        bottom: -6px;
        margin-left: -7px;
        left: 50%;
      }
      .ol-tooltip-static:before {
        border-top-color: #ffcc33;
      }
</style>
