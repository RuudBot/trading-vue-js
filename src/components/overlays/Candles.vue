<script>
// Renedrer for candlesticks + volume (optional)
// It can be used as the main chart or an indicator

import Overlay from '../../mixins/overlay.js'
import { layout_cnv } from '../js/layout_cnv.js'
import Candle from '../primitives/candle.js'
import Volbar from '../primitives/volbar.js'

export default {
    name: 'Candles',
    mixins: [Overlay],
    methods: {
        meta_info() {
            return { author: 'C451', version: '1.1.0' }
        },

        draw(ctx) {

            // If data === main candlestick data
            // render as main chart:
            if (this.$props.sub === this.$props.data) {
                var cnv = {
                    candles: this.$props.layout.candles,
                    volume: this.$props.layout.volume,
                }
            // Else, as offchart / onchart indicator:
            } else {
                cnv = layout_cnv(this)
            }

            if (this.show_volume) {
                for (var v of cnv.volume) {
                    new Volbar(this, ctx, v)
                }
            }

            for (var c of cnv.candles) {
                new Candle(this, ctx, c)
            }

        },
        use_for() { return ['Candles'] },

        // When added as offchart overlay
        y_range() {
            return [
                Math.max(...this.$props.sub.map(x => x[2])),
                Math.min(...this.$props.sub.map(x => x[3]))
            ]
        }
    },

    // Define internal setting & constants here
    computed: {
        sett() {
            return this.$props.settings
        },
        show_volume() {
            return 'showVolume' in this.sett ?
                this.sett.showVolume : true
        },
        colorCandleUp() {
            return this.sett.colorCandleUp ||
            this.$props.colors.colorCandleUp
        },
        colorCandleDw() {
            return this.sett.colorCandleDw ||
            this.$props.colors.colorCandleDw
        },
        colorWickUp() {
            return this.sett.colorWickUp ||
            this.$props.colors.colorWickUp
        },
        colorWickDw() {
            return this.sett.colorWickDw ||
            this.$props.colors.colorWickDw
        },
        colorWickSm() {
            return this.sett.colorWickSm ||
            this.$props.colors.colorWickSm
        },
        colorVolUp() {
            return this.sett.colorVolUp ||
            this.$props.colors.colorVolUp
        },
        colorVolDw() {
            return this.sett.colorVolDw ||
            this.$props.colors.colorVolDw
        }
    },
    data() {
        return {}
    }

}
</script>
