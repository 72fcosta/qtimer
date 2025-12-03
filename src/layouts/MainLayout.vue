<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated class="bg-primary text-white">
      <q-toolbar>
        <q-icon name="hourglass_empty" color="white" class="q-mr-sm" />
        <q-toolbar-title>QTimer</q-toolbar-title>
        <div class="text-caption">Ciclos configuráveis</div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script setup lang="ts"></script>
