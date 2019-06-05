<template>
    <table class="table-condensed">
        <thead>
        <tr>
            <th class="prev available" @click="$emit('prevMonth')"><span/></th>
            <th
                    v-if="showDropdowns"
                    :colspan="showWeekNumbers ? 6 : 5"
                    class="month"
            >
                <div class="row mx-1">
                    <select v-model="month" class="monthselect col">
                        <option v-for="(m, idx) in months" :key="m" :value="idx">{{m}}</option>
                    </select>
                    <select v-model="year" class="yearselect col">
                        <option v-for="y in years" :key="y" :value="y">{{y}}</option>
                    </select>
                </div>
            </th>
            <th v-else colspan="5" class="month">{{monthName}} {{year}}</th>
            <th class="next available" @click="$emit('nextMonth')"><span/></th>
        </tr>
        </thead>
        <tbody>
        <tr>
            <th v-if="showWeekNumbers" class="week">{{locale.weekLabel}}</th>
            <th v-for="(weekDay, index) in locale.daysOfWeek" :key="`weekDay-${index}`">{{weekDay}}</th>
        </tr>
        <tr
                v-for="(dateRow, index) in calendar"
                :key="index"
        >
            <td v-if="showWeekNumbers && (index%7 || index===0)" class="week">
                {{dateRow[0] | weeknumber}}
            </td>
            <slot name="date-slot" v-for="(date, idx) in dateRow">
                <td
                        :class="dayClass(date)"
                        @click="$emit('dateClick', date)"
                        @mouseover="$emit('hoverDate', date)"
                        :key="idx"
                >
                    {{date | dateNum}}
                </td>
            </slot>
        </tr>
        </tbody>
    </table>
</template>

<script>
  import moment from 'moment'

  export default {
    name: 'calendar',
    props: {
      monthDate: Date,
      locale: Object,
      start: Date,
      end: Date,
      minDate: Date,
      maxDate: Date,
      showDropdowns: {
        type: Boolean,
        default: false,
      },
      showWeekNumbers: {
        type: Boolean,
        default: false,
      },

      rangeMode:{
        type: String,
        default: "day"
        // "day, week, month"
      }
    },
    methods: {
      dayClass (date) {
        let dt = new Date(date)
        dt.setHours(0, 0, 0, 0)
        let start = new Date(this.start)
        start.setHours(0, 0, 0, 0)
        let end = new Date(this.end)
        end.setHours(0, 0, 0, 0)

        //Attain extrema of week or month from the start:date
        const minExt =  moment(start).startOf('week'); 
        const maxExt =  moment(start).endOf('week');
        const minExtMth =  moment(start).startOf('month');
        const maxExtMth =  moment(start).endOf('month');
        return {
          off: date.month() !== this.month,
          weekend: date.isoWeekday() > 5,
          today: dt.setHours(0, 0, 0, 0) == new Date().setHours(0, 0, 0, 0),
          active: this.rangeMode === "day" && (dt.setHours(0, 0, 0, 0) == new Date(this.start).setHours(0, 0, 0, 0) || dt.setHours(0, 0, 0, 0) == new Date(this.end).setHours(0, 0, 0, 0)),
          'in-range': (this.rangeMode === "day" && (dt >= start && dt <= end)) || (this.rangeMode === "week") && (dt >= minExt && dt <= maxExt) || (this.rangeMode === "month")  && (dt >= minExtMth && dt <= maxExtMth),
          'start-date': this.rangeMode === "day" && dt.getTime() === start.getTime(),
          'end-date': this.rangeMode === "day" && dt.getTime() === end.getTime(),
          disabled: (this.minDate && moment(dt).startOf("day").isBefore(moment(this.minDate).startOf("day")))
            || (this.maxDate && moment(dt).startOf("day").isAfter(moment(this.maxDate).startOf("day"))),
        }
      },
      getStartDay (daysInLastMonth, dayOfWeek, lastYear, lastMonth){
        let startDay = daysInLastMonth - dayOfWeek + this.locale.firstDay + 1
        if (startDay > daysInLastMonth)
          startDay -= 7
        if (dayOfWeek === this.locale.firstDay)
          startDay = daysInLastMonth - 6;
        // hack!!! need a more elegant fix
        const curDate = moment([lastYear, lastMonth, startDay, 12, 0, 0]);
        if (!curDate.isValid())
          startDay -=1
        // hack!!! need a more elegant fix
        return startDay;
      }
    },
    computed: {
      monthName () {
        return this.locale.monthNames[this.monthDate.getMonth()]
      },
      year: {
        get () {
          return this.monthDate.getFullYear()
        },
        set (value) {
          this.$emit('change-month', {
            month: this.month,
            year: value,
          });
        }
      },
      month: {
        get () {
          return this.monthDate.getMonth()
        },
        set (value) {
          this.$emit('change-month', {
            month: value,
            year: this.year,
          });
        }
      },
      calendar () {
        const month = this.month
        const year = this.monthDate.getFullYear()
        //TODO:swap out this line
        // const daysInMonth = new Date(year, month, 0).getDate()
        const daysInMonth = moment(
          year.toString() + "-" + ((month + 1) > 9 ? (month + 1).toString() : "0" + (month + 1).toString()),
          "YYYY-MM").daysInMonth()
        const firstDay = new Date(year, month, 1)
        // delete this line isn't necessary
        // let lastDay = new Date(year, month, daysInMonth)
        const lastMonth = moment(firstDay).subtract(1, 'month').month()
        const lastYear = moment(firstDay).subtract(1, 'month').year()
        const daysInLastMonth = moment([lastYear, lastMonth]).daysInMonth()

        const dayOfWeek = firstDay.getDay()

        let calendar = []

        for (let i = 0; i < 6; i++) {
          calendar[i] = [];
        }
        // this block needs to be evaluated
        // let startDay = daysInLastMonth - dayOfWeek + this.locale.firstDay + 1
        // if (startDay > daysInLastMonth)
        //   startDay -= 7
        //
        // if (dayOfWeek === this.locale.firstDay)
        //   startDay = daysInLastMonth - 6;
        // this block needs to be evaluated
        const startDay = this.getStartDay(daysInLastMonth, dayOfWeek, lastYear, lastMonth)

        let curDate = moment([lastYear, lastMonth, startDay, 12, 0, 0]);
        for (let i = 0, col = 0, row = 0; i < 6 * 7; i++, col++, curDate = moment(curDate).add(1, 'day')) {
          if (i > 0 && col % 7 === 0) {
            col = 0;
            row++;
          }
          calendar[row][col] = curDate.clone()
          curDate.hour(12);
        }

        return calendar
      },
      months () {
        let y = this.maxDate.getFullYear() - this.minDate.getFullYear();
        if (y < 2) {
          // get months
          let months = [];
          if (y < 1) {
            for (let i = this.minDate.getMonth(); i <= this.maxDate.getMonth(); i++) {
              months.push(i);
            }
          } else {
            for (let i = 0; i <= this.maxDate.getMonth(); i++) {
              months.push(i);
            }
            for (let i = this.minDate.getMonth(); i < 12; i++) {
              months.push(i);
            }
          }

          // do filter
          if (months.length > 0) {
            return this.locale.monthNames.filter((m, index) => {
              return months.findIndex(i => i === index) > -1;
            });
          }
        }
        return this.locale.monthNames;
      },
      years () {
        let values = []
        let count = 0
        // for (let i = this.minDate.getFullYear(); i <= this.maxDate.getFullYear() && count <= 20; i++) {
        for (let i = this.start.getFullYear(); count <= 20; i++) {
          count ++
          values.push(i - 10)
        }
        return values;
      }
    },
    filters: {
      dateNum (value) {
        return value.date()
      },
      weeknumber (value) {
        return value.week()
      }
    }
  }
</script>

<style scoped lang="scss">
    th,td {
        padding: 2px;
        background-color: white;
    }

    td.today {
        font-weight: bold;
    }

    td.disabled {
        pointer-events: none;
        background-color: #eee;
        border-radius: 0;
        opacity: 0.6;
    }

    @function str-replace($string, $search, $replace: "") {
        $index: str-index($string, $search);

        @if $index {
            @return str-slice($string, 1, $index - 1) + $replace + str-replace(str-slice($string, $index + str-length($search)), $search, $replace);
        }

        @return $string;
    }

    $carousel-control-color: #ccc !default;
    $viewbox: '-2 -2 10 10';
    $carousel-control-prev-icon-bg: str-replace(url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='#{$viewbox}'%3E%3Cpath d='M5.25 0l-4 4 4 4 1.5-1.5-2.5-2.5 2.5-2.5-1.5-1.5z'/%3E%3C/svg%3E"), "#", "%23") !default;
    $carousel-control-next-icon-bg: str-replace(url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='#{$viewbox}'%3E%3Cpath d='M2.75 0l-1.5 1.5 2.5 2.5-2.5 2.5 1.5 1.5 4-4-4-4z'/%3E%3C/svg%3E"), "#", "%23") !default;

    .fa {
        display: inline-block;
        width: 100%;
        height: 100%;
        background: transparent no-repeat center center;
        background-size: 100% 100%;
        fill: $carousel-control-color;
    }

    .prev, .next {
        &:hover {
            background-color: transparent !important;
        }

        .fa:hover {
            opacity: 0.6;
        }
    }

    .chevron-left {
        width: 16px;
        height: 16px;
        display: block;
        background-image: $carousel-control-prev-icon-bg;
    }

    .chevron-right {
        width: 16px;
        height: 16px;
        display: block;
        background-image: $carousel-control-next-icon-bg;
    }
</style>
