<template>
  <div id="app">

    <nav v-if="hasViews" class="main-nav">
      <Burger></Burger>
    </nav>
    <Sidebar>
      <ul class="sidebar-panel-nav">
        <li>
          <a href="/">Everything</a>
        </li>
        <li v-for="view in this.serverInfo.views" :key="view.id">
          <a :href="get_view_url(view)">{{ view.name }}</a>
        </li>
      </ul>
    </Sidebar>

    <div id="builds">
      <vue-headful :title="title" />
      <section v-if="errored && !docker">
        <p>
          The Duck server could not be reached at {{ this.address }}. Retrying...
          <pulse-loader color="#DDDDDD" size="8px" style="text-align: left;" />
        </p>
      </section>
      <section v-else-if="errored && docker">
        <p>
          The Duck server could not be reached. Retrying...
          <pulse-loader color="#DDDDDD" size="8px" style="text-align: left;" />
        </p>
      </section>
      <section v-else-if="loading">
        <p>Loading...</p>
      </section>
      <section v-else>
        <builds :builds="allBuilds" />
      </section>
      <vue-progress-bar style="z-index:-1"></vue-progress-bar>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import builds from "./components/Builds.vue";
import Burger from "./components/Menu/Burger.vue";
import Sidebar from "./components/Menu/Sidebar.vue";
import PulseLoader from "vue-spinner/src/PulseLoader.vue";

export default {
  name: "App",
  components: {
    Builds: builds,
    Burger,
    Sidebar,
    PulseLoader
  },
  data() {
    return {
      address: process.env.VUE_APP_MY_DUCK_SERVER,
      docker: process.env.VUE_APP_MY_DUCK_SERVER == "",
      view: this.$route.query.view,
      serverInfo: null,
      builds: null,
      loading: true,
      errored: false
    };
  },
  computed: {
    hasViews() {
      if(this.serverInfo != undefined) {
        return this.serverInfo.views.length > 0;
      }
      return false;
    },
    title() {
      if (this.serverInfo == null) {
        return "Duck";
      } else {
        if (this.view != undefined) {
          for (var i=0; i < this.serverInfo.views.length; i++) {
              if (this.serverInfo.views[i].slug === this.view) {
                  return this.serverInfo.views[i].name;
              }
          }
        }
        return this.serverInfo.title;
      }
    },
    allBuilds() {
      return this.builds
        .slice()
        .sort((a, b) => (a.started < b.started ? 1 : -1));
    }
  },
  methods: {
    get_view_url: function(view) {
      return "/?view=" + view.slug
    },
    loadData: function() {
      this.$Progress.start();

      let address = this.address + "/api/builds";
      if (this.view != undefined) {
        address = address + "/view/" + this.view;
      }

      axios
        .get(address)
        .then(response => {
          this.builds = response.data;
          this.errored = false;
          this.$Progress.finish();

          if (this.serverInfo == null) {
            this.updateServerInfo();
          }
        })
        .catch(() => {
          this.errored = true;
          this.$Progress.fail();
        })
        .finally(() => (this.loading = false));
    },
    updateServerInfo: function() {
      axios
        .get(this.address + "/api/server")
        .then(response => {
          this.serverInfo = response.data;
        })
        .catch(() => {
          this.serverInfo = null;
        });
    }
  },
  mounted() {
    this.loadData();
    setInterval(
      function() {
        this.loadData();
      }.bind(this),
      3000
    );
  }
};
</script>

<style scoped>
#title {
  padding-top: 5px;
  padding-bottom: 5px;
  padding-left: 8px;
}
#builds {
  padding-top: 20px;
  padding-left: 20px;
  padding-right: 20px;
}
.main-nav {
  display: flex;
  width: 1px;
  float: right;
  justify-content: space-between;
  padding-right: 64px;
  padding-top: 16px;
}
ul.sidebar-panel-nav {
  list-style-type: none;
}
ul.sidebar-panel-nav > li > a {
  color: #bbb;
  text-decoration: none;
  font-size: 1.5rem;
  display: block;
  padding-bottom: 0.5em;
}
ul.sidebar-panel-nav > li > a:hover {
  color: #fff;
  text-decoration: none;
  font-size: 1.5rem;
  display: block;
  padding-bottom: 0.5em;
}
</style>