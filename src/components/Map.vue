<template>
  <div id="mapContainer">
    <div id="titleContainer">
      <h1 id="title">人口自然增加率</h1>
      <p id="subtitle">2019年第4季</p>
    </div>
    <div id="map"></div>
    <div id="legend" v-show="showLegend">
      <div id="colorContainer">
        <div
          class="colors"
          v-for="(color, index) in colorPalette"
          :key="`color-${index}`"
          :style="{
            'background-color': color
          }"
        ></div>
      </div>
      <div id="textContainer">
        <div
          class="legendText"
          v-for="(number, index) in renderedBreaks"
          :key="`number-${index}`"
        >
          {{ number }}‰
        </div>
      </div>
    </div>
    <div
      id="legendButton"
      v-show="isMobile"
      @click="toggleLegend"
      :class="{ 'move-right': showLegend, 'move-left': !showLegend }"
    >
      <div
        :class="{ 'arrow-right': showLegend, 'arrow-left': !showLegend }"
      ></div>
    </div>
    <div id="infoBox" v-html="result"></div>
  </div>
</template>

<script>
import * as topojson from "topojson-client";
import mapboxgl from "mapbox-gl";
import { json } from "d3-fetch";
import bbox from "@turf/bbox";

export default {
  name: "Map",
  data() {
    return {
      result: "",
      colorPalette: [],
      showLegend: true,
      legendBreaks: [],
      renderedBreaks: [],
      zoomLevel: 0,
      zoomBreaks: []
    };
  },
  computed: {
    isMobile() {
      return (
        typeof window.orientation !== "undefined" ||
        navigator.userAgent.indexOf("IEMobile") !== -1
      );
    }
  },
  watch: {
    zoomLevel: function(val) {
      this.renderedBreaks =
        val > this.zoomBreaks[1][1]
          ? this.legendBreaks.map(d => d * 3)
          : val > this.zoomBreaks[0][1]
          ? this.legendBreaks.map(d => d * 2)
          : this.legendBreaks;
    }
  },
  methods: {
    toggleLegend() {
      this.showLegend = !this.showLegend;
    }
  },
  async mounted() {
    const _vue = this;

    if (_vue.isMobile) _vue.showLegend = false;

    const theme = "FLD1";

    let selected = {};

    const files = ["county", "town", "village"];

    _vue.zoomBreaks = _vue.isMobile
      ? [
          [0, 7],
          [7, 11],
          [11, 24]
        ]
      : [
          [0, 8],
          [8, 12],
          [12, 24]
        ];

    const results = await Promise.all(
      files.map(d => json(`./data/${d}4326.json`))
    );

    const [county, town, village] = results.map((d, i) =>
      topojson.feature(d, d.objects[`${files[i]}4326`])
    );

    [county, town, village].forEach((o, i) => {
      o.features = o.features.map(d => ({
        ...d,
        id: d.properties[
          Object.keys(d.properties).filter(d =>
            new RegExp(`^${files[i].substr(0, 1).toUpperCase()}.+ID$`).test(d)
          )[0]
        ].replace(/-/, "")
      }));
    });

    const blankStyle = {
      version: 8,
      name: "Blank",
      center: [0, 0],
      zoom: 0,
      sources: {},
      layers: [
        {
          id: "background",
          type: "background",
          paint: {
            "background-color": "white"
          }
        }
      ],
      id: "blank",
      glyphs: "http://fonts.openmaptiles.org/{fontstack}/{range}.pbf"
    };
    const center = [120.8, 24];

    const map = new mapboxgl.Map({
      container: "map",
      style: blankStyle,
      center: center,
      minZoom: _vue.isMobile ? 6 : 7,
      zoom: _vue.isMobile ? 6 : 7
    });

    const colors = [
      "#b2182b",
      "#d6604d",
      "#f4a582",
      "#fddbc7",
      "#d1e5f0",
      "#92c5de",
      "#4393c3",
      "#2166ac"
    ];
    _vue.colorPalette = colors;

    let breaks = Array.from(Array(8).keys()).map(
      d => (d + 1) / (colors.length / 2) - 1
    );

    function createPolygon(d, i) {
      let colorScheme = colors
        .map((d, j) => [d, breaks[j] * (i + 1)])
        .reduce((a, b) => [...a, ...b]);

      colorScheme = colorScheme.slice(0, 15);
      breaks = breaks.slice(0, 7);

      _vue.legendBreaks = breaks;
      _vue.renderedBreaks = breaks;

      const source = files[i];
      const id = `${source}Polygon`;

      map.addSource(source, {
        type: "geojson",
        data: d
      });

      map.addLayer({
        id: id,
        source: source,
        minzoom: _vue.zoomBreaks[i][0],
        maxzoom: _vue.zoomBreaks[i][1],
        type: "fill",
        paint: {
          "fill-color": ["step", ["get", theme], ...colorScheme],
          "fill-opacity": 0.8,
          "fill-outline-color": "rgba(0, 0, 0, 0)"
        }
      });
    }

    function createBoundary(d, i) {
      const source = files[i];
      const id = `${source}Boundary`;

      map.addLayer({
        id: id,
        source: source,
        minzoom: _vue.zoomBreaks[i + 1][0],
        maxzoom: _vue.zoomBreaks[i + 1][1],
        type: "line",
        paint: {
          "line-color": "rgba(0, 0, 0, 1)",
          "line-width": 1
        }
      });
    }

    function createSelectedFeature(d, i) {
      const source = files[i];
      const id = `${source}Selected`;

      map.addLayer({
        id: id,
        source: source,
        minzoom: _vue.zoomBreaks[i][0],
        maxzoom: _vue.zoomBreaks[i][1],
        type: "line",
        paint: {
          "line-color": [
            "case",
            ["boolean", ["feature-state", "ta"], false],
            "rgba(0, 0, 0, 1)",
            "rgba(0, 0, 0, 0)"
          ],
          "line-width": 2
        }
      });
    }

    function getBounds(feature) {
      const boundingBox = bbox(feature);
      const sw = new mapboxgl.LngLat(boundingBox[0], boundingBox[1]);
      const ne = new mapboxgl.LngLat(boundingBox[2], boundingBox[3]);
      const bounds = new mapboxgl.LngLatBounds(sw, ne);

      return bounds;
    }

    function addListener(event) {
      map.on(event, e => {
        const target =
          map.queryRenderedFeatures(e.point).length > 0
            ? map.queryRenderedFeatures(e.point)[0]
            : { properties: {} };

        const info = target.properties;
        const result = `<p>${info.COUNTY || ""}${info.TOWN ||
          ""}${info.VILLAGE || ""}${
          theme in info
            ? `</p><p>${info[theme]}‰</p>`
            : "<p>請點擊／指向地圖</p>"
        }`;

        if ("COUNTY" in info) {
          if (event === "click") {
            const bounds = getBounds(target.geometry);
            if (!info.VILLAGE) map.fitBounds(bounds, { padding: 40 });
          }
          if (Object.keys(selected).length !== 0) {
            map.setFeatureState(selected, { ta: false });
          }
          map.setFeatureState(target, { ta: true });
          selected = target;
        } else {
          if (Object.keys(selected).length !== 0) {
            map.setFeatureState(selected, { ta: false });
          }
          selected = {};
        }
        _vue.result = result;
      });
    }

    map.on("zoom", () => {
      _vue.zoomLevel = map.getZoom();
    });

    ["click", "mousemove"].forEach(d => addListener(d));

    map.on("load", () => {
      [county, town, village].forEach((d, i) => createPolygon(d, i));
      [county, town, village].forEach((d, i) => createSelectedFeature(d, i));
      [county, town].forEach((d, i) => createBoundary(d, i));
    });
  }
};
</script>

<style lang="scss">
#map {
  width: 100vw;
  height: 100vh;
}

#mapContainer {
  @extend #map;
  position: relative;
}

#legend {
  position: absolute;
  right: 5%;
  bottom: 10%;
  width: 100px;
  height: 300px;
  display: flex;
  padding: 10px 0;
  #colorContainer {
    height: 100%;
    flex-grow: 1;
    display: flex;
    flex-direction: column-reverse;
    justify-content: space-between;
    .colors {
      width: 20px;
      height: 20px;
      margin: 0 auto;
      opacity: 0.8;
    }
  }

  #textContainer {
    text-align: center;
    height: 100%;
    flex-grow: 1;
    display: flex;
    flex-direction: column-reverse;
    justify-content: space-evenly;
  }
}

#legendButton {
  position: absolute;
  right: 0;
  bottom: calc(10% + 300px + 20px - 50px);
  width: 10px;
  height: 50px;
}

#infoBox {
  position: absolute;
  left: 10%;
  top: 17%;
  background-color: rgba(255, 255, 255, 0.8);
  padding: 0 10px;
  p {
    font-size: 1.5rem;
    margin-top: 0.5rem;
    margin-bottom: 0.5rem;
  }
}

#titleContainer {
  padding: 0 10px;
  position: absolute;
  left: 10%;
  top: 5%;
  margin: 0;
  #title {
    font-size: 2.5rem;
    margin: 0;
  }
  #subtitle {
    margin: 0;
    font-size: 1.5rem;
  }
}

@media all and (max-width: 480px) {
  #titleContainer {
    #title {
      font-size: 1.5rem;
    }
    #subtitle {
      font-size: 1rem;
    }
  }

  #infoBox p {
    font-size: 1rem;
  }

  #legend {
    right: 0;
    width: 75px;
  }

  .legendText {
    font-size: 0.75rem;
  }

  #titleContainer {
    left: 5%;
  }

  #infoBox {
    left: 5%;
  }
}

#titleContainer,
#infoBox,
#legend {
  z-index: 1;
}

#legendButton {
  z-index: 2;
}

#map,
#mapContainer {
  z-index: 0;
}

@mixin arrow {
  width: 0;
  height: 0;
  border-top: 5px solid transparent;
  border-bottom: 5px solid transparent;
  transform: translateY(22.5px);
  margin: auto;
}

.arrow-right {
  @include arrow;
  border-left: 5px solid black;
}

.arrow-left {
  @include arrow;
  border-right: 5px solid black;
}

.move-left {
  transform: translateX(0px);
}

.move-right {
  transform: translateX(-75px);
}

#titleContainer,
#legend,
#infoBox {
  pointer-events: none;
}
</style>
