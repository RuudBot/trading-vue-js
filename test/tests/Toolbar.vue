<template>
<trading-vue :data="chart" :width="this.width" :height="this.height"
        title-txt="The King"
        :toolbar="true"
        :color-back="colors.colorBack"
        :color-grid="colors.colorGrid"
        :color-text="colors.colorText">
</trading-vue>
</template>

<script>
import TradingVue from '../../src/TradingVue.vue'
import Data from '../data/data_tools.json'
import Utils from '../../src/stuff/utils.js'
import DataCube from '../../src/helpers/datacube.js'

export default {
    name: 'Toolbar',
    description: 'Toolbar & drawing tools',
    components: {
        TradingVue
    },
    methods: {
        onResize(event) {
            this.width = window.innerWidth
            this.height = window.innerHeight - 50
        }
    },
    mounted() {
        window.addEventListener('resize', this.onResize)
        this.onResize()
        window.DataCube = this.chart
    },
    beforeDestroy() {
        window.removeEventListener('resize', this.onResize)
    },
    data() {
        return {
            chart: new DataCube(Data),
            width: window.innerWidth,
            height: window.innerHeight,
            colors: {
                colorBack: '#fff',
                colorGrid: '#eee',
                colorText: '#333'
            }
        }
    }
}
</script>

<style>

</style>
