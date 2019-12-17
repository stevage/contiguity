<template lang="pug">
#map.absolute.absolute--fill
</template>

<script>
import mapboxgl from 'mapbox-gl';
import U from 'mapbox-gl-utils';

import { EventBus } from './EventBus';
import { getFeatures, layer } from './sharedMapApi';
import boundingBox from 'geojson-bounding-box';
// const turf = require('@turf/turf');
// import turf from 'turf';
export default {
    data: () => ({
        points: undefined,
        selectedId: undefined,
        selected: undefined,
        colors: {},
    }),
    async mounted() {
        // replace this Mapbox access token with your own
        mapboxgl.accessToken = 'pk.eyJ1Ijoic3RldmFnZSIsImEiOiJjazNmNGV5enAwMTF1M2tuejhtc2twcXo5In0.mLPrYIYJ2FiFZ3KMqVIj6w';
        const map = new mapboxgl.Map({
            container: 'map',
            center: [144.96, -37.81],
            zoom: 14,
            style: 'mapbox://styles/mapbox/light-v9',
        });
        U.init(map, mapboxgl);
        window.map = map;
        window.app.Map = this;
        
        // window.Map = this;

        map.U.onLoad(async () => {
            map.U.addGeoJSON('points');
            map.U.addCircle('points-circles', 'points', {
                // circleColor: 'hsl(330,100%,70%)',
                // circleStrokeColor: 'hsl(330,100%,40%)',
                circleStrokeColor: 'black',
                circleColor: ['get', 'color'],
                // circleStrokeWidth: 3,
                circleStrokeWidth: ['case',
                    ['to-boolean', ["feature-state", "selected"]], 8,
                    3],

                circleRadius: { stops: [[10,3], [12, 10]] }
            });
            map.U.addSymbol('points-labels', 'points', {
                textField: '{name}',
                textColor: 'hsl(330,100%,30%)',
                textAnchor: 'left',
                textOffset: [1,0]
            });
            map.U.addGeoJSON('new-point');
            map.U.addCircle('new-point-circle', 'new-point', {
                circleColor: 'transparent',
                circleStrokeColor: 'hsl(120,80%,30%)',
                circleStrokeWidth: 5,
                circleRadius: { stops: [[10,3], [12, 10]] }
            });

            map.U.addGeoJSON('boundaries');
            map.U.addLine('boundaries-line', 'boundaries', {
                lineColor: ['get','color'],//'hsl(330, 90%,50%)'
                lineWidth: 4,
                lineOffset: 2
            });
            map.U.addFill('boundaries-fill','boundaries', {
                fillColor: ['get','color'],
                fillOpacity:0.15
            });
            
            map.U.hoverPointer('points-circles');
            map.on('click', 'points-circles', e => {
                if (NewFeature.mode === '') {
                    console.log(e);
                    window.FeatureInfo.feature = e.features[0];
                }
            });
            map.on('click', e => {
                if (window.NewFeature.mode === 'locating') {
                    EventBus.$emit('Map-clickLocate', e.lngLat);
                    map.U.setData('new-point', {
                        type: 'Point',
                        coordinates: [e.lngLat.lng, e.lngLat.lat]
                    });
                }
            });

            EventBus.$on('NewFeature-mode', mode => {
                if (mode === 'locating') {
                    map.getCanvas().style.cursor = 'crosshair'

                } else if (mode === '') {
                    map.getCanvas().style.cursor = ''

                    map.U.setData('new-point', { type: 'FeatureCollection', features: []});
                } else {
                    map.getCanvas().style.cursor = ''
                }
            });
            EventBus.$on('NewFeature-saved', newFeature => {
                this.points.features.push(newFeature);
                this.pointsUpdated(map, this.points);

            });
            EventBus.$on('delete-feature', id => {
                this.points.features = this.points.features.filter(f => f.id !== id);
                this.pointsUpdated(map, this.points);
                this.selected = undefined;
            });        
            EventBus.$on('update-feature', feature => {
                Object.assign(this.points.features.find(f => f.id === feature._id), feature);
                this.pointsUpdated(map, this.points);
            });        
            EventBus.$on('select-feature', feature => {
                if (this.selectedId) {
                    map.setFeatureState({ source: 'points', id: this.selectedId }, { selected: false });
                }
                this.selectedId = feature.id;
                this.selected = this.points.features.find(p => p.id === feature.id);
                window.NewFeature.cloneFeature = JSON.parse(JSON.stringify(this.selected));

                map.setFeatureState({ source: 'points', id: feature.id }, { selected: true });

                // EventBus.$emit('start-clone', feature);
            });        

            if (layer) {
                this.points = await getFeatures();

                if (this.points) {
                    map.fitBounds(boundingBox(this.points), { padding: 60 });
                }
                this.pointsUpdated(map, this.points);
            }
        });

    },
    methods: {
        pointsUpdated(map, points) {
            points.features.forEach(point => point.properties.color = getColor(point.properties.name, this.colors));
            map.U.setData('points', points);
            updateBoundaries(map, this.points);
        }
    }

}


function getColor(value, colors) {
    if (colors[value]) {
        return colors[value];
    }
    const n = Object.keys(colors).length;
    colors[value] = `hsl(${(n * 80) % 360}, ${100 - n * 3}%, ${70 - n * 2}%)`;
    console.log(`${value} => ${colors[value]}`);
    return colors[value];

}

function updateBoundaries(map, points) {
    // would like to allow a non-rectangular border
    // const border = turf.convex(points);
    // const bbox = turf.bbox(points);
    const bbox = turf.bbox(turf.transformScale(turf.bboxPolygon(turf.bbox(points)), 1.5));

    let boundaries = turf.voronoi(points, { bbox });
    boundaries.features.forEach((b, i) => b.properties = JSON.parse(JSON.stringify(points.features[i].properties)));


    const ids = {};
    for (const b of boundaries.features) {
        ids[b.properties.name] = true;
    }
    for (const key of Object.keys(ids)) {
        // boundaries.features.forEach(b => b.properties.dissolve = b.properties.name === key ? 'yes' : 'no')
        boundaries = turf.dissolve(boundaries, { propertyName: 'name' });
    }

    console.log(boundaries);
    map.U.setData('boundaries', boundaries);
    map.fitBounds(bbox, { padding: 60 });

}
import 'mapbox-gl/dist/mapbox-gl.css';

</script>

<style scoped>
</style>