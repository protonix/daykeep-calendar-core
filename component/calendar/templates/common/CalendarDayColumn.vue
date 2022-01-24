<template>
  <div :class="columnCss">
    <!-- underlying cells -->
    <template
      v-for="thisHour in 24"
    >
      <div
        :key="thisHour"
        :style="getCellStyle"
        :id="getDayHourId(eventRef, workingDate, thisHour - 1)"
        @drop="drop($event)" @dragover="allowDrop($event)"
      >
        <div class="calendar-day-time-content"></div>
      </div>
      <div
        v-if="showHalfHours"
        :style="getCellStyle"
        :key="thisHour + '-half'"
      >
        <div class="calendar-day-time-content-half"></div>
      </div>
    </template>

    <!-- events -->
    <template v-if="dateEvents.length > 0">
      <template v-for="eventObject in dateEvents">
        <div
          v-if="!eventObject.start.isAllDay && !eventObject.timeSpansMultipleDays"
          :key="makeDT(workingDate).toISODate() + getEventIdString(eventObject)"
          :class="calculateDayEventClass(eventObject)"
          :style="calculateDayEventStyle(eventObject)"
        >
          <calendar-event
            draggable="true" style="cursor: move;" :id="'drag_' + eventObject.id "
            :event-object="eventObject"
            :event-ref="eventRef"
            :calendar-locale="calendarLocale"
            :calendar-timezone="calendarTimezone"
            :prevent-event-detail="preventEventDetail"
            :allow-editing="allowEditing"
            render-style="doubleLine"
          />
        </div>
      </template>
    </template>

    <!-- current time -->
    <div
      class="current-time-line"
      :style="timePosition"
    ></div>

  </div>
</template>

<script>
  import CalendarEvent from './CalendarEvent'
  import {
    CalendarMixin,
    CalendarDayColumnTemplateMixin
  } from '../../mixins'

  export default {
    name: 'CalendarDayColumn',
    components: {
      CalendarEvent
    },
    mixins: [
      CalendarMixin,
      CalendarDayColumnTemplateMixin
    ],
    methods: {
      allowDrop(ev) {
        ev.preventDefault();
      },
      drop(ev) {
        ev.preventDefault();
        var data = ev.dataTransfer.getData("text");
        var new_div = ev.target.id; // calendar-2022-01-25-hour-1
        new_div = new_div.replace('calendar-', '');
        var parts = new_div.split('-hour-');
        var event_date = parts[0];
        var event_time = parts[1];
        if( event_time <= 9 ) { event_time = '0' + event_time; }
        event_time = event_time + ":00:00";
        var event_id = data.replace('drag_', '');

        ev.target.appendChild(document.getElementById(data));

        let old_event_date = localStorage.getItem("event_date");
        let old_event_time = localStorage.getItem("event_time");
        //console.log( old_event_date, old_event_time );

        let params = {
          event_id: event_id,
          event_date: event_date, event_time: event_time,
          old_event_date: old_event_date, old_event_time: old_event_time
        }
        //console.log( params )
        //............ API REQUEST .............
        this.api_query('drag-event', 'POST', params, (data) => {
          console.dir(data);
        }, (error) => {
          console.error(error);
        });
        //........ END - API Request
      },

      /** ==============================================================
       * ================== Send data via API ==========================
       * ============================================================== */
      api_query(url, method, filter, callback, onError) {

        if( !method || method == '') {  method = 'GET';  }

        //............ API REQUEST .............

        let requestOptions = {
          method: method,
          headers: {
            "Content-Type": "application/json",
            Authorization: `bearer ${localStorage.getItem("user_token")}`,
          }
        };
        if( method == 'POST' ) {
          requestOptions.body = JSON.stringify(filter);
        }
        fetch(localStorage.getItem("api_url") + 'api/' + url, requestOptions)
          .then(async response => {
            const data = await response.json();
            if (!response.ok) {
              const error = (data && data.message) || response.status;
              return Promise.reject(error);
            } else {
              return callback(data);
            }
          })
          .catch(error => {
            console.error('Error:', error);
            return onError(error);
          });
        // ....... END - API Request
      },


    }
  }
</script>

<style lang="stylus">
  /*@import '../../styles-common/calendar.vars.styl'*/

  .calendar-day
    position relative
    .calendar-day-cell-height
      height 5rem
      max-height 5rem
    .calendar-day-column-label
      //
    .calendar-day-column-content
      position relative
    .calendar-day-column-current
      background-color $currentDayBackgroundColor
    .calendar-day-column-weekend
      background-color $weekendDayBackgroundColor
    .calendar-day-time
      padding-right .5em
      border-right $borderOuter
    .calendar-day-time-content
      border-top $borderThin
    .calendar-day-time-content-half
      border-top $borderThinner
    .calendar-day-event-overlap
      margin-left 1px
      ::after
        position absolute
        top -1px
        left -1px
        width: calc(100% + 2px)
        height: calc(100% + 2px)
        content ''
        border-radius 5px
        border 1px solid white
        box-sizing border-box
    .calendar-day-event-overlap-first
      margin-left 0
    .current-time-line
      position absolute
      border 1px solid red
      width 100%

</style>
