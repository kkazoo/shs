<template>
  <confirm-popup :show="show" ok-text="Select" @cancel="cancel" @ok="done">
    <div class="time-picker">
      <div class="time-selector">
        <scroll-selector
          v-model="selectedTime.hour"
          :style="{ textAlign: 'right' }"
          :options="hours"
          font-size="1.5em"
        />
        <span class="colon">:</span>
        <scroll-selector v-model="selectedTime.minute" :options="minutes" font-size="1.5em" />
        <scroll-selector v-model="selectedTime.suffix" :options="suffixes" font-size="1.5em" />
      </div>

      <div class="separator">
        <div class="line" />
        <div class="text">or select an existing period</div>
        <div class="line" />
      </div>

      <div class="options">
        <div
          v-for="option in options"
          :key="`${option.text}-${option.time}`"
          class="option"
          :class="{ selected: option === selectedPeriod }"
          @click="choosePeriod(option)"
        >
          <div class="text">{{ option.text }}</div>
          <div class="time">
            {{ convertMilitaryTime(option.time) }}
            <span class="suffix">{{ getSuffix(option.time) }}</span>
          </div>
        </div>
      </div>

      <checkbox v-show="mode === 'period'" v-model="setOthersChecked" class="set-others">
        <div class="text">Also set periods with the same name in other schedule types</div>
      </checkbox>

      <div class="separator"><div class="line" /></div>
    </div>
  </confirm-popup>
</template>

<script lang="ts">
import ConfirmPopup from '@/components/ConfirmPopup.vue';
import ScrollSelector from '@/components/ScrollSelector.vue';
import Checkbox from '@/components/Checkbox.vue';
import { defineComponent } from 'vue';
import Bell from '@/utils/bell';

interface TimePickerPeriod {
  time: string;
  mode: string;
}

interface TimePickerOption {
 text: string;
 time: string;
 scheduleMode: string;
 name: string;
}

export default defineComponent({
  components: {
    ConfirmPopup,
    ScrollSelector,
    Checkbox,
  },
  data() {
    return {
      show: false,
      options: [] as TimePickerPeriod[],
      cancel: () => {},
      done: () => {},
      hours: Array(12).fill(0).map((_, i) => String(i + 1)) as string[], // generates [' 1', ' 2', ' 3' ..., '12']
      minutes: Array(60).fill(0).map((_, i) => i).map((val) => (val < 10 ? '0' : '') + val) as string[], // generates ['00', '01', '02', ..., '59']
      suffixes: ['AM', 'PM'] as ('AM' | 'PM')[],
      selectedTime: {
        hour: '1',
        minute: '00',
        suffix: 'AM',
      },
      mode: 'time', // can be 'time' or 'period'
      selectedPeriod: {} as TimePickerPeriod | null,
      setOthersChecked: true,
    };
  },
  computed: {
    time(): { time: string, mode: string, setOthers: boolean} | { time: string} | string {
      if (this.mode === 'period') {
        return this.selectedPeriod ? {
          ...this.selectedPeriod,
          setOthers: this.setOthersChecked,
        } : '0:00';
      }
      // we assume mode === 'time'
      return { time: this.getSelectedTime() };
    },
  },
  watch: {
    selectedTime: {
      handler() {
        if (this.mode !== 'time' && (!this.selectedPeriod || this.selectedPeriod.time !== this.getSelectedTime())) {
          this.mode = 'time';
          this.selectedPeriod = null;
        }
      },
      deep: true,
    },
  },
  methods: {
    pickTime(options = [] as TimePickerOption[], selectedTime = '1:00') {
      // options is an array of objects of the format { time: String ("12:45"), text: String }
      return new Promise((resolve, reject) => {
        this.options = options.slice(0); // make a copy to prevent modification of the original
        this.options.sort((a:TimePickerOption, b:TimePickerOption) => Bell.timeToNumber(a.time) - Bell.timeToNumber(b.time));
        this.setOthersChecked = true;

        this.show = true;

        this.setSelectedTime(selectedTime);

        this.cancel = () => {
          reject(Error('User Canceled'));
          this.show = false;
        };

        this.done = () => {
          resolve(this.time);
          this.show = false;
        };
      });
    },
    choosePeriod(period: TimePickerPeriod): void {
      this.selectedPeriod = period;
      this.setSelectedTime(period.time);
      this.mode = 'period';
    },
    getSelectedTime(): string {
      let hours = Number(this.selectedTime.hour);
      if (this.selectedTime.suffix === 'PM' && hours < 12) hours += 12;
      if (this.selectedTime.suffix === 'AM' && hours === 12) hours -= 12;

      return `${hours}:${this.selectedTime.minute}`;
    },
    setSelectedTime(time: string): void {
      let [hours, minutes] = time.split(':') as [string | number, string | number]; // eslint-disable-line prefer-const
      hours = Number(hours);

      const suffix = hours >= 12 ? 'PM' : 'AM';
      hours = (hours > 12) ? hours - 12 : hours;
      hours = (hours === 0) ? 12 : hours;

      this.selectedTime.hour = String(hours);
      this.selectedTime.minute = String(minutes);
      this.selectedTime.suffix = suffix;
    },
    convertMilitaryTime: Bell.convertMilitaryTime,
    getSuffix: Bell.getSuffix,
  },
});
</script>

<style lang="sass" scoped>
@import '@/styles/style.sass'

.time-picker
  margin-top: 15px

  .time-selector
    display: flex
    align-items: center
    margin: 0 35px
    justify-content: center

    .colon
      font-size: 1.5em

  .separator
    display: flex
    align-items: center
    margin: 15px 0

    .line
      flex: 1
      height: 1px
      background-color: #bbb

    .text
      color: #666
      font-size: .8em
      margin: 0 5px

  .options
    height: 200px
    overflow: auto
    +no-scrollbar
    padding: 0 35px

    .option
      display: flex
      justify-content: space-between
      border-radius: 100px
      border: var(--color) thin solid
      margin: 10px 10px
      padding: 5px 10px
      cursor: pointer
      min-width: 200px
      &.selected
        background-color: var(--color)
        color: var(--background)

      .suffix
        font-size: .75em

  .set-others .text
    width: 200px

</style>
