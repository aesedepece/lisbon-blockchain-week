<template>
  <div class="agenda">
    <div class="header">
      <h2 class="text">October 2021</h2>
      <Select
        :options="options"
        :default-option="selected"
        @update-selected="updateSelected"
      />
    </div>
    <div class="content">
      <div class="day-switch">
        <div
          v-for="day in days"
          :key="day.key"
          class="day"
          :class="{ current: currentDay === day.key }"
          @click="setCurrentDay(day.key)"
        >
          <p>{{ day.name }}</p>
          <p>{{ day.key }}</p>
        </div>
      </div>
      <div v-if="filteredEvents.length" class="events-list">
        <div
          v-for="event in filteredEvents"
          :key="event.title"
          class="row"
          :class="{ current: event.current }"
        >
          <div class="col svg-container" @click="makeEventFav(event.id)">
            <SvgIcon
              name="heart"
              :current-class="event.favorite ? 'fav' : 'default'"
            />
          </div>
          <p class="col">{{ event.startTime }} - {{ event.endTime }}</p>
          <div>
            <div class="trigger col" @click="showModal(event)">
              <p class="trigger title">{{ event.title }}</p>
            </div>
          </div>
          <div class="col categories">
            <p
              v-for="category in event.categories"
              :key="category"
              class="label"
              :class="{ current: event.current }"
            >
              {{ category }}
            </p>
          </div>
        </div>
      </div>
      <Empty v-else :day="currentDay" :category="selected.label" />
      <EventInfoModal
        v-if="currentEvent"
        :event="currentEvent"
        @close="closeModal"
      />
    </div>
  </div>
</template>

<script>
import { format } from 'date-fns'
import { fetchEvents } from '../fetchEvents'
import { EVENT_CATEGORIES, WEEK, EVENTS_URL } from '~/constants.js'

export default {
  data() {
    return {
      days: WEEK,
      options: EVENT_CATEGORIES,
      eventsUrl: EVENTS_URL,
      selected: { label: 'All' },
      filteredEvents: [],
      selectedEvents: [],
      currentDay: '18',
      currentEvent: null,
      isModalVisible: false,
      events: [],
      favorites: [],
    }
  },
  async fetch() {},
  computed: {
    defaultDay() {
      return format(new Date(1634544000000), 'dd-MM-yyyy')
    },
    keyDays() {
      return Object.keys(this.days)
    },
  },
  watch: {
    favorites() {
      this.setFavorites()
    },
    currentDay() {
      this.filterEvents(this.selectedEvents)
    },
  },
  async mounted() {
    // Load events from csv stored in github
    this.events = await fetchEvents(this.eventsUrl)
    // Load favorites from localStorage
    this.loadFavorites()
    // Add favorite key to the events stored as fav in localStorage
    this.setFavorites()
    this.setCurrentEvent()
    // Show events for the default day
    this.filterEvents()
  },
  methods: {
    loadFavorites() {
      this.favorites = JSON.parse(localStorage.getItem('favorites')) || []
    },
    setFavorites() {
      if (this.favorites) {
        this.events = this.events.map((event) => {
          return {
            ...event,
            favorite: this.favorites.find((favorite) => favorite === event.id),
          }
        })
        // Filter event to re-render favorites
        this.filterEvents()
      }
    },
    makeEventFav(id) {
      if (this.favorites.includes(id)) {
        this.favorites = this.favorites.filter((x) => x !== id)
      } else {
        this.favorites.push(id)
      }

      if (process.browser) {
        localStorage.setItem('favorites', JSON.stringify(this.favorites))
      }
      this.$nextTick(() => {
        this.updateSelected(this.selected)
      })
    },
    setCurrentDay(day) {
      this.currentDay = day
    },
    showModal(event) {
      document.body.style.top = `-${window.scrollY}px`
      document.body.style.position = 'fixed'
      this.$nextTick(() => {
        this.currentEvent = event
      })
    },
    closeModal() {
      this.currentEvent = null
      this.$nextTick(() => {
        const scrollY = document.body.style.top
        document.body.style.position = ''
        document.body.style.top = ''
        window.scrollTo(0, parseInt(scrollY || '0') * -1)
      })
    },
    getDate(timestamp) {
      return format(new Date(timestamp), 'dd-MM-yyyy')
    },
    getDay(timestamp) {
      return format(new Date(timestamp), 'dd')
    },
    setCurrentEvent() {
      this.events.map((event) => {
        if (
          event.startTimestamp < new Date().getTime() &&
          event.endTimestamp > new Date().getTime()
        ) {
          this.currentDay = this.getDay(event.endTimestamp)
          event.current = true
        } else {
          event.current = false
        }
        return event
      })
      this.selectedEvents = this.events
    },
    filterEvents(events) {
      const eventsToFilter = events || this.events
      const isTimeForEvent = this.days[this.currentDay].timestamp
      if (!isTimeForEvent) {
        this.currentDay = this.keyDays[0]
      }
      this.filteredEvents = eventsToFilter
        .filter((event) => {
          return (
            this.getDate(event.startTimestamp) ===
            this.getDate(this.days[this.currentDay].timestamp || 0)
          )
        })
        .sort((ev1, ev2) => ev1.startTimestamp - ev2.startTimestamp)
    },
    updateSelected(selectedOption) {
      this.selected = selectedOption
      if (this.selected.label === 'All') {
        this.selectedEvents = this.events
      } else if (this.selected.label === 'Favorites') {
        this.selectedEvents = this.events.filter((event) => {
          return event.favorite
        })
      } else {
        this.selectedEvents = this.events.filter((event) => {
          return event.categories
            .map((category) => category.toLowerCase())
            .includes(this.selected.label.toLowerCase())
        })
      }
      this.filterEvents(this.selectedEvents)
    },
  },
}
</script>

<style scoped lang="scss">
.current {
  color: $red;
  &.label {
    border: 1px solid $red;
  }
}
.agenda {
  display: grid;
  grid-template: max-content 1fr / 1fr;
  grid-row-gap: 34px;
  justify-content: center;
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid $grey;
    padding: 34px 0;
    font-family: JoystixMonospace, sans-serif;
  }
  .content {
    display: grid;
    grid-template: max-content 1fr / 1fr;
    grid-row-gap: 34px;
    .day-switch {
      display: flex;
      justify-content: space-around;
    }
    .row {
      display: grid;
      grid-template-columns: max-content auto auto 1fr;
      padding: 24px;
      font-size: 18px;
      grid-column-gap: 34px;
      align-items: center;
      justify-content: center;
      border-bottom: 1px solid $grey;
    }
    .svg-container {
      cursor: pointer;
    }
    .title {
      text-decoration: underline;
      cursor: pointer;
    }
    .col {
      margin: 16px;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;

      &.categories {
        justify-content: flex-end;
      }

      .label {
        border: 1px solid $black;
        width: max-content;
        padding: 4px 8px;
        margin: 8px;
      }
      .current {
        &.label {
          border: 1px solid $red;
        }
      }
    }
    .day {
      margin: 16px;
      font-size: 24px;
      cursor: pointer;
      text-align: center;
    }
  }
}
@media (max-width: 780px) {
  .agenda {
    .content {
      grid-template-columns: auto 1fr;
      grid-column-gap: 24px;
      justify-content: flex-start;

      .day-switch {
        flex-direction: column;
        justify-content: initial;
        position: sticky;
        top: 0;
        height: 100vh;
      }

      .events-list {
        .row {
          display: flex;
          flex-direction: column;

          .col {
            text-align: center;
            margin: 8px;

            &.categories {
              justify-content: center;
              margin-right: 0;
              margin-left: 0;
            }
          }
        }
      }
    }
  }
}
</style>
