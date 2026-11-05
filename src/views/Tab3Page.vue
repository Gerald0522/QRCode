
<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Crear publicación</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" class="ion-padding">

      <h2>Nueva publicación</h2>

      <ion-button
        expand="block"
        color="secondary"
        :disabled="escaneando"
        @click="iniciarEscaner"
      >
        Escanear código QR
      </ion-button>

      <div v-show="escaneando" class="ion-margin-top">
        <div id="lector-qr"></div>

        <ion-button
          expand="block"
          color="danger"
          @click="detenerEscaner"
        >
          Cerrar cámara
        </ion-button>
      </div>

      <p v-if="mensaje">{{ mensaje }}</p>

      <ion-list>
        <ion-item>
          <ion-input
            label="Título"
            label-placement="stacked"
            placeholder="Ingrese el título"
            v-model="titulo"
          ></ion-input>
        </ion-item>

        <ion-item>
          <ion-textarea
            label="Contenido"
            label-placement="stacked"
            placeholder="Ingrese el contenido"
            :auto-grow="true"
            v-model="contenido"
          ></ion-textarea>
        </ion-item>

        <ion-item>
          <ion-select
            label="Usuario"
            label-placement="stacked"
            placeholder="Seleccione un usuario"
            v-model="usuarioId"
          >
            <ion-select-option
              v-for="usuario in usuarios"
              :key="usuario.id"
              :value="usuario.id"
            >
              {{ usuario.name }}
            </ion-select-option>
          </ion-select>
        </ion-item>
      </ion-list>

      <ion-button
        expand="block"
        class="ion-margin-top"
        :disabled="guardando"
        @click="crearPost"
      >
        Crear publicación
      </ion-button>

      <ion-card v-if="postCreado">
        <ion-card-header>
          <ion-card-title>Publicación creada</ion-card-title>
        </ion-card-header>

        <ion-card-content>
          <p><strong>ID:</strong> {{ postCreado.id }}</p>
          <p><strong>Título:</strong> {{ postCreado.title }}</p>
          <p><strong>Contenido:</strong> {{ postCreado.body }}</p>
          <p><strong>Usuario:</strong> {{ postCreado.userId }}</p>
        </ion-card-content>
      </ion-card>

    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount } from 'vue';
import { onIonViewWillLeave } from '@ionic/vue';
import { Html5Qrcode } from 'html5-qrcode';
import type { Usuarios } from '@/interfaces/usuarios';

import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonList,
  IonItem,
  IonInput,
  IonTextarea,
  IonSelect,
  IonSelectOption,
  IonButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardContent
} from '@ionic/vue';

interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

const usuarios = ref<Usuarios[]>([]);

const titulo = ref('');
const contenido = ref('');
const usuarioId = ref<number | null>(null);

const postCreado = ref<Post | null>(null);
const mensaje = ref('');
const escaneando = ref(false);
const guardando = ref(false);

let lectorQR: Html5Qrcode | null = null;
let iniciando = false;
let procesandoQR = false;

// Cargar usuarios
function cargarUsuarios() {
  fetch('https://jsonplaceholder.typicode.com/users')
    .then(response => {
      if (!response.ok) {
        throw new Error('Error al cargar usuarios');
      }
      return response.json();
    })
    .then((json: Usuarios[]) => {
      usuarios.value = json;
    })
    .catch(error => {
      console.error(error);
      mensaje.value = 'No se pudieron cargar los usuarios.';
    });
}

cargarUsuarios();

// Iniciar cámara
async function iniciarEscaner() {
  if (lectorQR || iniciando) return;

  mensaje.value = '';
  iniciando = true;
  escaneando.value = true;
  procesandoQR = false;

  try {
    lectorQR = new Html5Qrcode('lector-qr');

    const camaras = await Html5Qrcode.getCameras();

    if (camaras.length === 0) {
      throw new Error('No se encontró ninguna cámara');
    }

    const camara = camaras[0];

    await lectorQR.start(
      camara.id,
      {
        fps: 10,
        qrbox: {
          width: 250,
          height: 250
        }
      },
      (textoQR: string) => {
        leerQR(textoQR);
      },
      () => {
        // Ignorar cuadros sin códigos QR
      }
    );

  } catch (error) {
    console.error(error);
    mensaje.value = 'No se pudo abrir la cámara.';
    await detenerEscaner();
  } finally {
    iniciando = false;
  }
}

// Leer los datos del QR

async function leerQR(textoQR: string) {
  if (procesandoQR) return;

  try {
    console.log('QR detectado:', textoQR);

    const lineas = textoQR.split('\n');

    const title = lineas.find(linea =>
      linea.trim().toLowerCase().startsWith('title:')
    )?.split(':').slice(1).join(':').trim();

    const body = lineas.find(linea =>
      linea.trim().toLowerCase().startsWith('body:')
    )?.split(':').slice(1).join(':').trim();

    const user = lineas.find(linea =>
      linea.trim().toLowerCase().startsWith('user:')
    )?.split(':').slice(1).join(':').trim();

    const id = Number(user);

    if (!title || !body || !user || !Number.isInteger(id)) {
      throw new Error('El QR no contiene los datos correctos');
    }

    if (!usuarios.value.some(usuario => usuario.id === id)) {
      throw new Error('El usuario no existe');
    }

    procesandoQR = true;

    titulo.value = title;
    contenido.value = body;
    usuarioId.value = id;

    await detenerEscaner();

    mensaje.value = 'Formulario completado correctamente';

  } catch (error) {
    console.error(error);

    mensaje.value = error instanceof Error
      ? error.message
      : 'Error al leer el QR';
  }
}


// Detener cámara
async function detenerEscaner() {
  const lector = lectorQR;
  lectorQR = null;

  if (lector) {
    try {
      if (lector.isScanning) {
        await lector.stop();
      }
      lector.clear();
    } catch (error) {
      console.error('Error al cerrar cámara:', error);
    }
  }

  escaneando.value = false;
}

// Crear publicación
function crearPost() {
  if (
    !titulo.value.trim() ||
    !contenido.value.trim() ||
    usuarioId.value === null
  ) {
    mensaje.value = 'Complete todos los campos.';
    return;
  }

  guardando.value = true;
  mensaje.value = '';

  fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify({
      title: titulo.value,
      body: contenido.value,
      userId: usuarioId.value
    }),
    headers: {
      'Content-type': 'application/json; charset=UTF-8'
    }
  })
    .then(response => {
      if (!response.ok) {
        throw new Error('Error al crear publicación');
      }
      return response.json();
    })
    .then((json: Post) => {
      postCreado.value = json;

      mensaje.value = 'Publicación creada correctamente.';

      titulo.value = '';
      contenido.value = '';
      usuarioId.value = null;
    })
    .catch(error => {
      console.error(error);
      mensaje.value = 'No se pudo crear la publicación.';
    })
    .finally(() => {
      guardando.value = false;
    });
}

// Cerrar cámara al salir del Tab
onIonViewWillLeave(() => {
  void detenerEscaner();
});

onBeforeUnmount(() => {
  void detenerEscaner();
});
</script>

<style scoped>
#lector-qr {
  width: 100%;
  max-width: 400px;
  margin: 15px auto;
  overflow: hidden;
  border-radius: 10px;
}
</style>
