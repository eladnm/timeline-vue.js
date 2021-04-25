<template>
  <div class="home">
    <v-col>
      <v-row>
        <h1 class="font-weight-bold">
          Activities View
        </h1>
      </v-row>
      <v-row>
        <h2>Timeline</h2>
        <v-btn to="/version2" small style="margin-left:10%;">Version2</v-btn>
        <List apiUrl="http://localhost:3000/activities/v1" />
      </v-row>
    </v-col>
  </div>
</template>

<script>
// @ is an alias to /src

import List from "../components/List";

export default {
  components: { List }
};
</script>
