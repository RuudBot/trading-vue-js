<template>
    <!-- Chart components combined together -->
    <div class="trading-vue-chart" :style="styles">
        <keyboard ref="keyboard"></keyboard>
        <grid-section v-for="(grid, i) in this._layout.grids"
            :key="grid.id"
            v-bind:common="section_props(i)"
            v-bind:grid_id="i"
            v-on:register-kb-listener="register_kb"
            v-on:remove-kb-listener="remove_kb"
            v-on:range-changed="range_changed"
            v-on:cursor-changed="cursor_changed"
            v-on:cursor-locked="cursor_locked"
            v-on:sidebar-transform="set_ytransform"
            v-on:layer-meta-props="layer_meta_props"
            v-on:custom-event="emit_custom_event"
            v-on:legend-button-click="legend_button_click"
            >
        </grid-section>
        <botbar v-bind="botbar_props"></botbar>
    </div>
</template>

<script>

import Context from '../stuff/context.js'
import Layout from './js/layout.js'
import Utils from '../stuff/utils.js'
import CursorUpdater from './js/updater.js'
import GridSection from './Section.vue'
import Botbar from './Botbar.vue'
import Keyboard from './Keyboard.vue'

export default {
    name: 'Chart',
    props: [
        'title_txt', 'data', 'width', 'height', 'font', 'colors',
        'overlays', 'tv_id', 'config', 'buttons', 'toolbar'
    ],
    components: {
        GridSection,
        Botbar,
        Keyboard
    },
    created() {

        // Context for text measurements
        this.ctx = new Context(this.$props)

        // Initial layout (All measurments for the chart)
        this.init_range()
        this.sub = this.subset()
        this._layout = new Layout(this)

        // Updates current cursor values
        this.updater = new CursorUpdater(this)

    },
    methods: {
        range_changed(r) {
            // Overwite & keep the original references
            Utils.overwrite(this.range, r)
            const sub = this.subset()
            Utils.overwrite(this.sub, sub)
            this.update_layout()
            this.$emit('range-changed', r)
        },
        goto(t) {
            const dt = this.range[1] - this.range[0]
            this.range_changed([t - dt, t])
        },
        setRange(t1, t2) {
            this.range_changed([t1, t2])
        },
        cursor_changed(e) {
            this.updater.sync(e)
        },
        cursor_locked(state) {
            if (this.cursor.scroll_lock && state) return
            this.cursor.locked = state
        },
        calc_interval() {
            if (this.ohlcv.length < 2) return
            this.interval = Utils.detect_interval(this.ohlcv)
        },
        set_ytransform(s) {
            let obj = this.y_transforms[s.grid_id] || {}
            Object.assign(obj, s)
            this.$set(this.y_transforms, s.grid_id, obj)
            this.update_layout()
            Utils.overwrite(this.range, this.range)
        },
        default_range() {
            const dl = this.$props.config.DEFAULT_LEN
            const ml = this.$props.config.MINIMUM_LEN + 0.5
            const l = this.ohlcv.length - 1

            if (this.ohlcv.length < 2) return
            if (this.ohlcv.length < dl) {
                var s = 0, d = ml
            } else {
                s = l - dl, d = 0.5
            }

            Utils.overwrite(this.range, [
                this.ohlcv[s][0] - this.interval * d,
                this.ohlcv[l][0] + this.interval * ml
            ])
        },
        subset() {
            return Utils.fast_filter(
                this.ohlcv,
                this.range[0] - this.interval,
                this.range[1]
            )
        },
        common_props() {
            return {
                title_txt: this.$props.title_txt,
                layout: this._layout,
                sub: this.sub,
                range: this.range,
                interval: this.interval,
                cursor: this.cursor,
                colors: this.$props.colors,
                font: this.$props.font,
                y_ts: this.y_transforms,
                tv_id: this.$props.tv_id,
                config: this.$props.config,
                buttons: this.$props.buttons
            }
        },
        overlay_subset(source) {
            return source.map(d => ({
                type: d.type,
                name: d.name,
                data: Utils.fast_filter(
                    d.data,
                    this.range[0] - this.interval,
                    this.range[1]
                ),
                settings: d.settings || this.settings_ov
            }))
        },
        section_props(i) {
            return i === 0 ?
                this.main_section : this.sub_section
        },
        init_range() {
            this.calc_interval()
            this.default_range()
        },
        layer_meta_props(d) {
            // TODO: check reactivity when layout is changed
            if (!(d.grid_id in this.layers_meta)) {
                this.$set(this.layers_meta, d.grid_id, {})
            }
            this.$set(this.layers_meta[d.grid_id],
                Utils.get_num_id(d.layer_id), d)

            // Rerender
            this.update_layout()
        },
        emit_custom_event(d) {
            this.$emit('custom-event', d)
        },
        update_layout(clac_tf) {
            if (clac_tf) this.calc_interval()
            const lay = new Layout(this)
            Utils.copy_layout(this._layout, lay)
        },
        legend_button_click(event) {
            this.$emit('legend-button-click', event)
        },
        register_kb(event) {
            if (!this.$refs.keyboard) return
            this.$refs.keyboard.register(event)
        },
        remove_kb(event) {
            if (!this.$refs.keyboard) return
            this.$refs.keyboard.remove(event)
        }
    },
    computed: {
        // Component-specific props subsets:
        main_section() {
            let p = Object.assign({}, this.common_props())
            p.data = this.overlay_subset(this.onchart)
            p.data.push({
                type: this.chart.type || 'Candles',
                main: true,
                data: this.sub,
                settings: this.chart.settings || this.settings_ohlcv
            })
            p.overlays = this.$props.overlays
            return p
        },
        sub_section() {
            let p = Object.assign({}, this.common_props())
            p.data = this.overlay_subset(this.offchart)
            p.overlays = this.$props.overlays
            return p
        },
        botbar_props() {
            let p = Object.assign({}, this.common_props())
            p.width = p.layout.botbar.width
            p.height = p.layout.botbar.height
            p.rerender = this.rerender
            return p
        },
        offsub() {
            return this.overlay_subset(this.offchart)
        },
        // Datasets: candles, onchart, offchart indicators
        ohlcv() {
            return this.$props.data.ohlcv || this.chart.data || []
        },
        chart() {
            return this.$props.data.chart || {}
        },
        onchart() {
            return this.$props.data.onchart || []
        },
        offchart() {
            return this.$props.data.offchart || []
        },
        styles() {
            let w = this.$props.toolbar ? this.$props.config.TOOLBAR : 0
            return { 'margin-left': `${w}px` }
        }
    },
    data() {
        return {
            // Current data slice
            sub: [],

            // Time range
            range: [],

            // Candlestick interval
            interval: 0,

            // Crosshair states
            cursor: {
                x: null, y: null, t: null, y$: null,
                grid_id: null, locked: false, values: {},
                scroll_lock: false
            },

            // A trick to re-render botbar
            rerender: 0,

            // Layers meta-props (changing behaviour)
            layers_meta: {},

            // Y-transforms (for y-zoom and -shift)
            y_transforms: {},

            // Default OHLCV settings (when using DataStructure v1.0)
            settings_ohlcv: {},

            // Default overlay settings
            settings_ov: {}
        }
    },
    watch: {
        width() {
            this.update_layout()
        },
        height() {
            this.update_layout()
        },
        colors() {
            Utils.overwrite(this.range, this.range)
        },
        data: {
            handler: function(n, p) {
                if (!this.sub.length) this.init_range()
                const sub = this.subset()
                // Fix Infinite loop warn, when the subset is empty
                // TODO: Consider removing 'sub' from data entirely
                if (this.sub.length || sub.length) {
                    Utils.overwrite(this.sub, sub)
                }
                this.update_layout(Utils.data_changed(n, p))
                Utils.overwrite(this.range, this.range)
                this.cursor.scroll_lock = !!n.scrollLock
                if (n.scrollLock && this.cursor.locked) {
                    this.cursor.locked = false
                }
                // TODO: update legend values for overalys
                this.rerender++
            },
            deep: true
        }
    }
}

</script>
