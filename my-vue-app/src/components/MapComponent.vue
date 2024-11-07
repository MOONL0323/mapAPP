<template>
  <div>
    <div>
      平行线数量：<input type="number" v-model="lineCount" />
      平行线间隔（米）：<input type="number" v-model="lineSpacing" />
    </div>
    <button @click="saveCoordinates">保存坐标</button>
    <div id="map" class="map"></div>
  </div>
</template>

<script>
import 'ol/ol.css';
import { Map, View } from 'ol';
import TileLayer from 'ol/layer/Tile';
import XYZ from 'ol/source/XYZ';
import VectorSource from 'ol/source/Vector';
import VectorLayer from 'ol/layer/Vector';
import { Point, LineString } from 'ol/geom';
import Feature from 'ol/Feature';
import { fromLonLat, toLonLat } from 'ol/proj';

export default {
  data() {
    return {
      map: null,
      source: new VectorSource(),
      points: [],
      lineCount: 10,
      lineSpacing: 10
    };
  },
  mounted() {
    this.initMap();
  },
  methods: {
    initMap() {
      this.map = new Map({
        target: 'map',
        layers: [
          new TileLayer({
            source: new XYZ({
              url: 'http://online{0-3}.map.bdimg.com/onlinelabel/?qt=tile&x={x}&y={y}&z={z}&styles=pl&scaler=1&p=1',
              maxZoom: 18
            })
          }),
          new VectorLayer({
            source: this.source
          })
        ],
        view: new View({
          center: fromLonLat([116.397389, 39.908722]), // 北京天安门
          zoom: 10
        })
      });

      this.map.on('click', this.handleMapClick);
    },
    handleMapClick(evt) {
      const coordinate = evt.coordinate;
      this.points.push(coordinate);

      const pointFeature = new Feature(new Point(coordinate));
      this.source.addFeature(pointFeature);

      if (this.points.length === 2) {
        this.drawMainLine();
        this.drawParallelLines();
      }
    },
    drawMainLine() {
      const line = new LineString(this.points);
      const lineFeature = new Feature(line);
      this.source.addFeature(lineFeature);
    },
    drawParallelLines() {
      const lineCount = parseInt(this.lineCount);
      const lineSpacing = parseFloat(this.lineSpacing);

      const mainLine = new LineString(this.points);
      const coordinates = mainLine.getCoordinates();

      const dx = coordinates[1][0] - coordinates[0][0];
      const dy = coordinates[1][1] - coordinates[0][1];
      const length = Math.sqrt(dx * dx + dy * dy);
      const ux = -dy / length;
      const uy = dx / length;

      for (let i = 1; i <= lineCount; i++) {
        const offset = lineSpacing * i;

        const offsetCoordinates1 = coordinates.map(coord => [coord[0] + ux * offset, coord[1] + uy * offset]);
        const parallelLine1 = new Feature(new LineString(offsetCoordinates1));
        this.source.addFeature(parallelLine1);

        const offsetCoordinates2 = coordinates.map(coord => [coord[0] - ux * offset, coord[1] - uy * offset]);
        const parallelLine2 = new Feature(new LineString(offsetCoordinates2));
        this.source.addFeature(parallelLine2);
      }
    },
    saveCoordinates() {
      const features = this.source.getFeatures();
      const lines = features.filter(feature => feature.getGeometry().getType() === 'LineString');

      let csvContent = "A_lon,A_lat,B_lon,B_lat\n";
      lines.forEach(line => {
        const coords = line.getGeometry().getCoordinates();
        const coordA = toLonLat(coords[0]);
        const coordB = toLonLat(coords[coords.length - 1]);
        csvContent += `${coordA[0]},${coordA[1]},${coordB[0]},${coordB[1]}\n`;
      });

      const hiddenElement = document.createElement('a');
      hiddenElement.href = 'data:text/csv;charset=utf-8,' + encodeURI(csvContent);
      hiddenElement.target = '_blank';
      hiddenElement.download = 'lines.csv';
      hiddenElement.click();
    }
  }
};
</script>

<style>
.map {
  width: 100%;
  height: 80vh;
}
</style>