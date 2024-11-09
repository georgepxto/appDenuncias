<template>
  <v-app :theme="theme">
    <v-main>
      <v-container>
        <v-app-bar :elevation="1">
          <template v-slot:prepend> </template>

          <v-app-bar-title href="./">Denuncias Anonimas</v-app-bar-title>

          <template v-slot:append>
            <v-btn href="./">Login como Parceiro</v-btn>
            <v-btn
              :prepend-icon="
                theme === 'light' ? 'mdi-weather-sunny' : 'mdi-weather-night'
              "
              slim
              @click="onClick"
            ></v-btn>
          </template>
        </v-app-bar>
      </v-container>
      <router-view />
    </v-main>

    <v-footer border> 2024 — Denuncias, LTDA</v-footer>
  </v-app>
</template>

<script setup>
import { ref } from "vue";

const theme = ref("dark");

function onClick() {
  theme.value = theme.value === "light" ? "dark" : "light";
}
</script>
