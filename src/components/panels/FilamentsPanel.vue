<template>
    <panel
        v-if="klipperReadyForGui && tools.length"
        :icon="mdiPrinter3dNozzleHeat"
        :title="$t('Panels.FilamentsPanel.Headline')"
        :collapsible="true"
        card-class="filaments-panel">
        <v-card-text class="pb-0">
            <v-row dense align="center">
                <v-col cols="4">
                    <v-text-field
                        v-model.number="temp"
                        type="number"
                        :label="$t('Panels.FilamentsPanel.Temperature')"
                        suffix="°C"
                        :disabled="running"
                        outlined
                        dense
                        hide-details />
                </v-col>
                <v-col>
                    <v-chip-group v-model="selected" multiple column active-class="primary">
                        <v-chip v-for="tool in tools" :key="tool" :value="tool" :disabled="running" filter small>
                            T{{ tool }}
                            <span class="ml-1" :title="$t('Panels.FilamentsPanel.Sensors')">
                                <v-icon v-for="(present, i) in sensors(tool)" :key="i" x-small>
                                    {{ present ? mdiCircle : mdiCircleOutline }}
                                </v-icon>
                            </span>
                        </v-chip>
                    </v-chip-group>
                </v-col>
            </v-row>
            <div v-if="running" class="text-caption mt-2">
                {{ $t(`Panels.FilamentsPanel.${job.mode}`) }}: {{ progress }}
            </div>
        </v-card-text>
        <v-card-actions class="px-4 pb-3">
            <v-btn small :disabled="!canStart" @click="start('FILAMENTS_UNLOAD')">
                <v-icon left small>{{ mdiArrowUpBold }}</v-icon>
                {{ $t('Panels.FilamentsPanel.Unload') }}
            </v-btn>
            <v-btn small :disabled="!canStart" @click="start('FILAMENTS_LOAD')">
                <v-icon left small>{{ mdiArrowDownBold }}</v-icon>
                {{ $t('Panels.FilamentsPanel.Load') }}
            </v-btn>
            <v-spacer />
            <v-btn v-if="running" small color="error" @click="doSend('FILAMENTS_STOP')">
                <v-icon left small>{{ mdiStop }}</v-icon>
                {{ $t('Panels.FilamentsPanel.Stop') }}
            </v-btn>
        </v-card-actions>
    </panel>
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import ControlMixin from '@/components/mixins/control'
import Panel from '@/components/ui/Panel.vue'
import { mdiArrowDownBold, mdiArrowUpBold, mdiCircle, mdiCircleOutline, mdiPrinter3dNozzleHeat, mdiStop } from '@mdi/js'

// _FILAMENTS macro variables: jobs = {tool: [extruder, mm moved, ticks waited]} for the tools still in progress
interface FilamentsJob {
    mode: string
    temp: number
    jobs: Record<string, [string, number, number]>
}

@Component({
    components: { Panel },
})
export default class FilamentsPanel extends Mixins(BaseMixin, ControlMixin) {
    mdiArrowDownBold = mdiArrowDownBold
    mdiArrowUpBold = mdiArrowUpBold
    mdiCircle = mdiCircle
    mdiCircleOutline = mdiCircleOutline
    mdiPrinter3dNozzleHeat = mdiPrinter3dNozzleHeat
    mdiStop = mdiStop

    temp = 240
    selected: number[] = []

    get tools(): number[] {
        return this.$store.state.printer.toolchanger?.tool_numbers ?? []
    }

    get job(): FilamentsJob | undefined {
        return this.$store.state.printer['gcode_macro _FILAMENTS']
    }

    get running(): boolean {
        return !!this.job?.mode
    }

    get canStart(): boolean {
        return !this.printerIsPrinting && !this.running && this.selected.length > 0 && this.temp > 0
    }

    get progress(): string {
        const temp = this.job?.temp ?? 0
        return Object.entries(this.job?.jobs ?? {})
            .map(([tool, [extruder, moved]]) => {
                const hot = (this.$store.state.printer[extruder]?.temperature ?? 0) >= temp - 5
                const upper = this.sensors(Number(tool))[0]
                let state = `${moved} mm`
                if (!hot) state = this.$t('Panels.FilamentsPanel.Heating').toString()
                else if (this.job?.mode === 'load' && !upper)
                    state = this.$t('Panels.FilamentsPanel.WaitingFilament').toString()
                return `T${tool} ${state}`
            })
            .join(' · ')
    }

    sensors(tool: number): boolean[] {
        return ['higher', 'lower'].map(
            (pos) => !!this.$store.state.printer[`filament_switch_sensor T${tool}_${pos}_switch`]?.filament_detected
        )
    }

    start(macro: string) {
        const tools = [...this.selected].sort((a, b) => a - b).join(',')
        this.doSend(`${macro} TOOLS=${tools} TEMP=${this.temp}`)
    }
}
</script>
