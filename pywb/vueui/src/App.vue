<template>
  <div class="app" :class="{expanded: showTimelineView || showFullView }" data-app="webrecorder-replay-app">
    <!-- Top navbar -->
    <nav
      class="navbar navbar-light navbar-expand-lg fixed-top top-navbar"
      :style="navbarBackgroundStyle">
      <a class="navbar-brand" href="/">
        <img :src="config.logoImg" alt="_('pywb logo')">
      </a>
      <form class="form-inline my-2 my-md-0" @submit="gotoUrl">
        <input id="theurl" type="text" :value="config.url"></input>
      </form>
      <button
        class="navbar-toggler btn btn-sm"
        type="button"
        data-toggle="collapse"
        data-target="#navbarCollapse"
        aria-controls="navbarCollapse"
        aria-expanded="false"
        aria-label="_('Toggle navigation')">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="navbarCollapse">
        <ul class="navbar-nav my-2 ml-3" id="toggles">
          <li class="nav-item active">
            <button
              class="btn btn-sm btn-outline-dark"
              :class="{active: showFullView}"
              :aria-pressed="(showFullView ? true : false)"
              @click="showFullView = !showFullView"
              :title="(showFullView ? _('Hide calendar') : _('Show calendar'))">
              <i class="far fa-calendar-alt"></i>
            </button>
          </li>
          <li class="nav-item ml-1">
            <button
              class="btn btn-sm btn-outline-dark"
              :class="{active: showTimelineView }"
              :aria-pressed="showTimelineView"
              @click="showTimelineView = !showTimelineView"
              :title="(showTimelineView ? _('Hide timeline') : _('Show timeline'))">
              <i class="far fa-chart-bar"></i>
            </button>
          </li>
          <li class="nav-item dropdown ml-1" v-if="localesAreSet">
            <button
              class="btn btn-sm btn-outline-dark dropdown-toggle"
              type="button"
              id="locale-dropdown"
              data-toggle="dropdown"
              aria-haspopup="true"
              aria-expanded="false"
              :title="_('Select language')">
              <i class="fas fa-globe-africa"></i>
            </button>
            <div class="dropdown-menu dropdown-menu-right" aria-labelledby="locale-dropdown">
              <a
                class="dropdown-item"
                v-for="(locPath, key) in config.allLocales"
                :key="key"
                :href="locPath + (currentSnapshot ? currentSnapshot.id : '*') + '/' + config.url">
                {{ key }}
              </a>
            </div>
          </li>
        </ul>
      </div>
    </nav>

    <!-- Capture title and date -->
    <nav
      class="navbar navbar-light justify-content-center title-nav fixed-top"
      id="second-navbar"
      :style="navbarBackgroundStyle"
      v-if="currentSnapshot">
      <span class="strong mr-1">
        {{_('Current Capture')}}: 
        <span class="ml-1" v-if="config.title">
          {{ config.title }}
        </span>
      </span>
      <span class="mr-1" v-if="config.title">,</span>
      {{currentSnapshot.getTimeDateFormatted()}}
    </nav>

    <!-- Timeline -->
    <div class="card timeline-wrap">
      <div class="card-body" v-if="currentPeriod && showTimelineView">
        <div class="row">
          <div class="col col-12">
            <TimelineBreadcrumbs
              :period="currentPeriod"
              @goto-period="gotoPeriod"
            ></TimelineBreadcrumbs>
          </div>
          <div class="col col-12 mt-2">
            <Timeline
              :period="currentPeriod"
              :highlight="timelineHighlight"
              :current-snapshot="currentSnapshot"
              :max-zoom-level="maxTimelineZoomLevel"
              @goto-period="gotoPeriod"
            ></Timeline>
          </div>
        </div>
      </div>    
    </div>

    <!-- Calendar -->
    <div class="card" v-if="currentPeriod && showFullView && currentPeriod.children.length">
      <div class="card-body">
        <CalendarYear
          :period="currentPeriod"
          :current-snapshot="currentSnapshot"
           @goto-period="gotoPeriod">
        </CalendarYear>
      </div>
    </div>
    
  </div>
</template>

<script>
import Timeline from "./components/Timeline.vue";
import TimelineBreadcrumbs from "./components/TimelineBreadcrumbs.vue";
import CalendarYear from "./components/CalendarYear.vue";

import { PywbSnapshot, PywbPeriod } from "./model.js";
import {PywbI18N} from "./i18n";

export default {
  name: "PywbReplayApp",
  //el: '[data-app="webrecorder-replay-app"]',
  data: function() {
    return {
      snapshots: [],
      currentPeriod: null,
      currentSnapshot: null,
      msgs: [],
      showFullView: true,
      showTimelineView: true,
      maxTimelineZoomLevel: PywbPeriod.Type.day,
      config: {
        title: "",
        initialView: {},
        allLocales: {}
      },
      timelineHighlight: false,
      locales: [],
    };
  },
  components: {Timeline, TimelineBreadcrumbs, CalendarYear},
  mounted: function() {
  },
  computed: {
    sessionStorageUrlKey() {
      // remove http(s), www and trailing slash
      return 'zoom__' + this.config.url.replace(/^https?:\/\/(www\.)?/, '').replace(/\/$/, '');
    },
    localesAreSet() {
      return Object.entries(this.config.allLocales).length > 0;
    },
    navbarBackgroundStyle() {
      return {
        '--navbar-background': `#${this.config.navbarBackgroundHash}`,
      }
    }
  },
  methods: {
    _(id, embeddedVariableStrings=null) {
      return PywbI18N.instance.getText(id, embeddedVariableStrings);
    },
    gotoPeriod: function(newPeriod, onlyZoomToPeriod) {
      if (this.timelineHighlight) {
        setTimeout((() => {
          this.timelineHighlight=false;
        }).bind(this), 3000);
      }
      // only go to snapshot if caller did not request to zoom only
      if (newPeriod.snapshot && !onlyZoomToPeriod) {
        this.gotoSnapshot(newPeriod.snapshot, newPeriod, true /* reloadIFrame */);
      } else {
        // save current period (aka zoom)
        // use sessionStorage (not localStorage), as we want this to be a very temporary memory for current page tab/window and no longer; NOTE: it serves when navigating from an "*" query to a specific capture and subsequent reloads
        if (window.sessionStorage) {
          window.sessionStorage.setItem(this.sessionStorageUrlKey, newPeriod.fullId);
        }
        // If new period goes beyond allowed max level
        if (newPeriod.type > this.maxTimelineZoomLevel) {
          this.currentPeriod = newPeriod.get(this.maxTimelineZoomLevel);
        } else {
          this.currentPeriod = newPeriod;
        }
      }
    },
    gotoSnapshot(snapshot, fromPeriod, reloadIFrame=false) {
      this.currentSnapshot = snapshot;

      // if the current period doesn't match the current snapshot, update it
      if (fromPeriod && !this.currentPeriod.contains(fromPeriod)) {
        const fromPeriodAtMaxZoomLevel = fromPeriod.get(this.maxTimelineZoomLevel);
        if (fromPeriodAtMaxZoomLevel !== this.currentPeriod) {
          this.currentPeriod = fromPeriodAtMaxZoomLevel;
        }
      }

      // update iframe only if the snapshot was selected from the calendar/timeline.
      // if the change originated from a user clicking a link in the iframe, emitting
      // snow-shapshot will only cause a flash of content
      if (reloadIFrame !== false) {
        this.$emit("show-snapshot", snapshot);
      }
      this.hideBannerUtilities();
    },
    gotoUrl(event) {
      event.preventDefault();
      const newUrl = document.querySelector("#theurl").value;
      if (newUrl !== this.url) {
        window.location.href = this.config.prefix + "*/" + newUrl;
      }
    },
    setData(/** @type {PywbData} data */ data) {

      // data-set will usually happen at App INIT (from parent caller)
      this.$set(this, "snapshots", data.snapshots);
      this.$set(this, "currentPeriod", data.timeline);

      // get last-saved current period from previous page/app refresh (if there was such)
      if (window.sessionStorage) {
        const currentPeriodId = window.sessionStorage.getItem(this.sessionStorageUrlKey);
        if (currentPeriodId) {
          const newCurrentPeriodFromStorage = this.currentPeriod.findByFullId(currentPeriodId);
          if (newCurrentPeriodFromStorage) {
            this.currentPeriod = newCurrentPeriodFromStorage;
          }
        }
      }

      // signal app is DONE setting and rendering data; ON NEXT TICK
      this.$nextTick(function isDone() {
        this.$emit('data-set-and-render-completed');
      }.bind(this));
    },
    setSnapshot(view) {
      // turn off calendar (aka full) view
      this.showFullView = false;

      // convert to snapshot object to support proper rendering of time/date
      const snapshot = new PywbSnapshot(view, 0);

      this.config.url = view.url;

      this.gotoSnapshot(snapshot);
    },
    setTimelineView() {
      this.showTimelineView = !this.showTimelineView;
      if (this.showTimelineView === true) {
        this.showFullView = false;
      }
    },
    hideBannerUtilities() {
      this.showFullView = false;
      this.showTimelineView = false;
    },
    updateTitle(title) {
      this.config.title = title;
    }
  }
};
</script>

<style>
  body {
    padding-top: 87px !important;
  }
  .app {
    font-family: Calibri, Arial, sans-serif;
    /*border-bottom: 1px solid lightcoral;*/
    width: 100%;
  }
  .app.expanded {
    height: 130px;
  }
  .full-view {
    /*position: fixed;*/
    /*top: 150px;*/
    left: 0;
  }
  .navbar {
    background-color: var(--navbar-background);
  }
  .top-navbar {
    z-index: 90;
    padding: 2px 16px 0 16px;
  }
  .title-nav {
    margin-top: 47px;
    z-index: 80;
  }
  #navbarCollapse {
    justify-content: right;
  }
  .iframe iframe {
    width: 100%;
    height: 80vh;
  }
  #theurl {
    width: 250px;
  }
  @media (min-width: 576px) {
    #theurl {
      width: 400px;
    }
  }
  @media (min-width: 768px) {
    #theurl {
      width: 500px;
    }
  }
  @media (min-width: 992px) {
    #theurl {
      width: 600px;
    }
  }
  @media (min-width: 1200px) {
    #theurl {
      width: 900px;
    }
  }
  #toggles {
    align-items: center;
  }
  .breadcrumb-row {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  div.timeline-wrap div.card {
    margin-top: 55px;
  }
  div.timeline-wrap div.card-body {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  div.timeline-wrap div.card-body div.row {
    width: 100%;
    align-items: center;
    justify-content: center;
  }
  .strong {
    font-weight: bold;
  }
</style>
