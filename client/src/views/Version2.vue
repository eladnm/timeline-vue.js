<template>
  <div class="home">
    <v-col>
      <v-row>
        <h1 class="font-weight-bold">
          Activities View 2
        </h1>
      </v-row>
      <v-row>
        <h2>Timeline</h2>
        <v-btn to="./" small style="margin-left:10%;">Version1</v-btn>
        <List apiUrl="http://localhost:3000/activities/v2" />
      </v-row>
    </v-col>
  </div>
</template>

<script>
import List from "../components/List";

export default {
  name: "Version2",
  components: { List },
  props: {
    msg: String
  }
};
</script>
