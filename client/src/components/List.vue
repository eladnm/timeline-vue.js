<template>
  <v-row class="text-left">
    <v-col class="mb-6" cols="6">
      <v-autocomplete
        clearable
        :items="names"
        placeholder="Search Timeline"
        v-model="search"
      ></v-autocomplete>
    </v-col>
    <v-col cols="12">
      <h4>Filter By:</h4>
      <v-chip @click="showAllItems" class="ma-2" label>
        <v-avatar v-if="selectedResources.length === 0" left>
          <v-icon>mdi-checkbox-marked-circle</v-icon>
        </v-avatar>
        All Items
      </v-chip>
      <v-chip
        v-for="(d, index) in resource"
        :key="index"
        class="ma-2"
        @click="addOrRemoveFilter(d)"
        label
      >
        <v-avatar v-if="selectedResources.find(el => el == d)" left>
          <v-icon>mdi-checkbox-marked-circle</v-icon>
        </v-avatar>

        {{ d }}
      </v-chip>
      <v-timeline dense>
        <template v-for="(list, group) in groupedData">
          <v-timeline-item color="red lighten-2" :key="group">
            {{ group.split(" ")[0] }}
          </v-timeline-item>
          <template v-for="d in list">
            <v-timeline-item fill-dot v-if="(d.visible = true)" :key="d.id" small left hide-dot>
              <v-card outlined class="d-flex">
                <v-avatar v-if="d.topic_data" size="80">
                  <img :src="'http://localhost:3000' + d.topic_data.icon_path" alt="" />
                </v-avatar>
                <v-card-text class="d-flex justify-space-between align-center">
                  <div class="d-flex flex-column text-left align-start font-weight-bold">
                    {{
                      capitalizeEveryWord(
                        d.topic_data.name + " " + d.resource_type.replaceAll("_", " ")
                      )
                    }}
                    <div class="d-flex justify-start align-start font-weight-regular">
                      {{ d.date }}
                    </div>
                  </div>

                  <div
                    class="text-right d-flex align-center"
                    :class="
                      checkSetting(d.resource_type, 'score')
                        ? 'justify-space-between'
                        : 'justify-end'
                    "
                    style="width: 25%"
                  >
                    <div
                      class="text-right d-flex align-center font-weight-bold"
                      v-if="d.score && checkSetting(d.resource_type, 'score')"
                    >
                      {{ "score" + " " + d.score + "/" + d.possible_score }}
                    </div>
                    <div class="d-flex align-center">
                      <v-btn
                        medium
                        color="red lighten-2"
                        dark
                        icon
                        class="mr-8"
                        @click="hideOnClick(d)"
                      >
                        <v-icon>mdi-eye-off</v-icon>
                        Hide
                      </v-btn>
                      <v-btn
                        small
                        v-if="checkSetting(d.resource_type, 'zoom')"
                        color="red lighten-2"
                        dark
                        @click="openDialog(d)"
                      >
                        View Work
                      </v-btn>
                    </div>
                  </div>
                </v-card-text>
              </v-card>
            </v-timeline-item>
          </template>
        </template>
        <v-row align="center" justify="center">
          <v-btn @click="loadMore" :loading="loading" :disabled="!hasMoreData || loading"
            >Load More</v-btn
          >
        </v-row>
      </v-timeline>
      <v-dialog v-model="dialog" width="500">
        <v-card>
          <v-card-title class="justify-center">
            <v-avatar v-if="selectedInfo && selectedInfo.topic_data" size="80">
              <img :src="'http://localhost:3000' + selectedInfo.topic_data.icon_path" alt="" />
            </v-avatar>
            <h3 class="text-center" v-if="selectedInfo && selectedInfo.topic_data">
              {{
                capitalizeEveryWord(
                  selectedInfo.topic_data.name +
                    " " +
                    selectedInfo.resource_type.replaceAll("_", " ")
                )
              }}
            </h3>
          </v-card-title>

          <v-card-text>
            <p class="text-center" v-if="selectedInfo && selectedInfo.date">
              {{ selectedInfo.date }}
            </p>
            <h3 v-if="selectedInfo && selectedInfo.comment">{{ selectedInfo.comment }}</h3>
            <p md6 v-if="selectedInfo && selectedInfo.score" style="margin-top:5%;">
              {{ "score" + " " + selectedInfo.score + "/" + selectedInfo.possible_score }}
            </p>
            <v-divider></v-divider>
          </v-card-text>

          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="primary" text @click="closeDialog">
              Close
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-col>
  </v-row>
</template>
<script>
import { format } from "date-fns/fp";

const axios = require("axios");
const capitalizeEveryWord = str => str.replace(/\b[a-z]/g, char => char.toUpperCase());

const groupBy = (arr, fn) =>
  arr.map(typeof fn === "function" ? fn : val => val[fn]).reduce((acc, val, i) => {
    acc[val] = (acc[val] || []).concat(arr[i]);
    return acc;
  }, {});
const flatten = (arr, depth = 1) =>
  arr.reduce((a, v) => a.concat(depth > 1 && Array.isArray(v) ? flatten(v, depth - 1) : v), []);
export default {
  name: "List",
  props: {
    apiUrl: {
      type: String,
      default: "http://localhost:3000/activities/v1"
    }
  },

  data: () => ({
    info: [],
    list: [],
    activities: [],
    dialog: false,
    search: "",
    perPage: 10,
    loading: false,
    paginatedData: [],
    selectedInfo: null,
    selectedResources: [],
    listOfObjects: {
      movie: {
        score: false,
        zoom: false
      },
      quiz: {
        score: true,
        zoom: true
      },
      easy_quiz: {
        score: true,
        zoom: true
      },
      make_a_map: {
        score: false,
        zoom: true
      },
      word_play: {
        score: false,
        zoom: true
      },
      related_reading: {
        score: false,
        zoom: false
      },
      make_a_movie: {
        score: false,
        zoom: true
      },
      challenge: {
        score: true,
        zoom: true
      },
      draw_about_it: {
        score: false,
        zoom: true
      }
    }
  }),
  mounted() {
    axios.get(this.apiUrl).then(response => {
      const list = response.data.map(el => {
        if (el.activities) {
          let activities = [];
          el.activities.forEach(activity => {
            activity = { resource_type: el.resource_type, ...activity };
            activity.date = this.formatDate(activity.d_created);
            activity.group = this.getMonthAndYear(activity.d_created);
            activity.visible = true;
            activities.push(activity);
          });
          return activities;
        } else {
          el.date = this.formatDate(el.d_created);
          el.group = this.getMonthAndYear(el.d_created);
          el.visible = true;
          return el;
        }
      });
      this.info = this.sortByDate(flatten(list, 2));
      this.list = this.info.slice(0, this.perPage);
    });
  },
  computed: {
    filteredInfo() {
      return this.list.filter(el => {
        if (!this.search) {
          return this.filterByResorce(el) && el.visible;
        }
        const name = el.topic_data ? el.topic_data.name.replaceAll("_", " ") : "";
        const fullname = capitalizeEveryWord(name + " " + el.resource_type.replaceAll("_", " "));
        return el.visible && !!fullname.match(this.search) && this.filterByResorce(el);
      });
    },
    names() {
      return this.info.map(el => {
        const name = el.topic_data ? el.topic_data.name.replaceAll("_", " ") : "";
        return capitalizeEveryWord(name + " " + el.resource_type.replaceAll("_", " "));
      });
    },
    resource() {
      return new Set(
        this.info.map(el => capitalizeEveryWord(el.resource_type.replaceAll("_", " ")))
      );
    },
    groupedData() {
      return groupBy(this.filteredInfo, "group");
    },
    hasMoreData() {
      return this.list.length < this.info.length;
    }
  },
  methods: {
    capitalizeEveryWord,
    openDialog(item) {
      this.selectedInfo = item;
      this.dialog = true;
    },
    closeDialog() {
      this.dialog = false;
      this.selectedInfo = null;
    },
    filterByResorce(item) {
      return this.selectedResources.length === 0
        ? true
        : this.selectedResources.filter(
            el => capitalizeEveryWord(item.resource_type.replaceAll("_", " ")) == el
          ).length > 0;
    },
    addOrRemoveFilter(item) {
      const el = this.selectedResources.findIndex(el => el == item);

      if (el !== -1) {
        this.$delete(this.selectedResources, el);
      } else {
        this.selectedResources.push(item);
      }
    },
    showAllItems() {
      this.selectedResources = [];
    },
    formatDate(unixTimestamp) {
      const milliseconds = unixTimestamp * 1000;
      const dateObject = new Date(milliseconds);

      return format("PPp", dateObject);
    },
    sortByDate(list) {
      return list.sort((a, b) => {
        if (a.d_created < b.d_created) return 1;
        if (a.d_created > b.d_created) return -1;
        return 0;
      });
    },
    checkSetting(resource, setting) {
      const res = this.listOfObjects[resource];
      if (!res) return false;
      return res[setting];
    },
    getMonthAndYear(unixTimestamp) {
      const milliseconds = unixTimestamp * 1000;
      const dateObject = new Date(milliseconds);
      return format("MMMM yyyy", dateObject);
    },
    loadMore() {
      this.loading = true;
      setTimeout(() => {
        const startIndex = this.list.length;
        const endIndex = startIndex + this.perPage;
        const nextData = this.info.slice(startIndex, endIndex);
        nextData.forEach(element => {
          this.list.push(element);
        });
        this.loading = false;
      }, 500);
    },
    hideOnClick(item) {
      const index = this.list.findIndex(el => el.id == item.id);
      if (index > -1) {
        item.visible = false;
        this.$set(this.list, index, item);
      }
    }
  }
};
</script>
