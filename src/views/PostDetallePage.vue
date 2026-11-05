
<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-back-button default-href="/tabs/tab2"></ion-back-button>
        </ion-buttons>

        <ion-title>Detalle de publicación</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">

      <ion-card v-if="post">
        <ion-card-header>
          <ion-card-title>
            {{ post.title }}
          </ion-card-title>

          <ion-card-subtitle>
            Publicación #{{ post.id }}
          </ion-card-subtitle>
        </ion-card-header>

        <ion-card-content>
          <p>{{ post.body }}</p>

          <p>Usuario: {{ post.userId }}</p>
        </ion-card-content>
      </ion-card>

    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import type { Posts } from '@/interfaces/posts';

import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonButtons,
  IonBackButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardSubtitle,
  IonCardContent
} from '@ionic/vue';

import { ref } from 'vue';
import { useRoute } from 'vue-router';

const route = useRoute();

const post = ref<Posts | null>(null);

function cargarPost() {

  fetch(`https://jsonplaceholder.typicode.com/posts/${route.params.id}`)
    .then(response => response.json())
    .then(json => {
      console.log(json);
      post.value = json;
    });

}

cargarPost();
</script>
