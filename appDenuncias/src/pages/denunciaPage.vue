<template>
  <div class="form-container">
    <div class="content-wrapper">
      <!-- Card principal -->
      <v-card class="pa-6">
        <v-card-title class="text-h5 mb-4">
          Formulário de Denúncia
        </v-card-title>

        <!-- Componente de Mapa -->
        <div class="mb-6">
          <div id="map" class="map-container"></div>
          <div class="map-controls mt-4">
            <div class="d-flex ga-2">
              <v-text-field
                v-model="searchQuery"
                @keyup.enter="searchLocation"
                label="Buscar endereço no mapa"
                prepend-inner-icon="mdi-magnify"
                variant="outlined"
                density="compact"
              ></v-text-field>
              <v-btn
                color="primary"
                :loading="loadingLocation"
                @click="getCurrentLocation"
              >
                <v-icon>mdi-crosshairs-gps</v-icon>
                Localização atual
              </v-btn>
            </div>
          </div>
        </div>

        <form @submit.prevent="submit">
          <div class="form-grid">
            <!-- Campo de localização da denúncia -->
            <v-text-field
              prepend-inner-icon="mdi-map-marker"
              v-model="localDenuncia.value.value"
              :error-messages="localDenuncia.errorMessage.value"
              label="Local da denúncia"
              variant="outlined"
            ></v-text-field>

            <!-- Campo de seleção de categoria -->
            <v-select
              v-model="select.value.value"
              :error-messages="select.errorMessage.value"
              :items="items"
              label="Selecione uma categoria"
              variant="outlined"
            ></v-select>

            <!-- Campo de título da denúncia -->
            <v-text-field
              v-model="name.value.value"
              :counter="10"
              :error-messages="name.errorMessage.value"
              label="Título da denúncia"
              variant="outlined"
            ></v-text-field>

            <!-- Campo de descrição da denúncia -->
            <v-textarea
              v-model="denuncia.value.value"
              :counter="600"
              :error-messages="denuncia.errorMessage.value"
              label="Descreva sua denúncia"
              variant="outlined"
              class="mb-4"
            ></v-textarea>

            <!-- Campo de upload de arquivo -->
            <v-file-input
              v-model="files"
              :show-size="1000"
              label="Envie um arquivo"
              placeholder="Escolha seu arquivo"
              prepend-icon="mdi-camera"
              variant="outlined"
              counter
              multiple
              class="mb-4"
            >
              <template v-slot:selection="{ fileNames }">
                <template
                  v-for="(fileName, index) in fileNames"
                  :key="fileName"
                >
                  <v-chip v-if="index < 2" class="me-2" size="small" label>
                    {{ fileName }}
                  </v-chip>

                  <span
                    v-else-if="index === 2"
                    class="text-overline text-grey-darken-3 mx-2"
                  >
                    +{{ files.length - 2 }} Arquivo(s)
                  </span>
                </template>
              </template>
            </v-file-input>

            <!-- Checkbox de termos e condições -->
            <div class="d-flex align-center mb-6">
              <v-checkbox
                v-model="checkbox.value.value"
                :error-messages="checkbox.errorMessage.value"
                label="Eu concordo com os"
                type="checkbox"
                value="1"
                color="surface-variant"
                variant="flat"
              ></v-checkbox>

              <!-- Dialog de Termos e Condições -->
              <v-dialog max-width="500">
                <template v-slot:activator="{ props: activatorProps }">
                  <span v-bind="activatorProps" class="terms-link">
                    Termos e Condições.
                  </span>
                </template>

                <template v-slot:default="{ isActive }">
                  <v-card title="Termos e condições">
                    <v-card-text>
                      Lorem ipsum dolor sit amet, consectetur adipiscing elit,
                      sed do eiusmod tempor incididunt ut labore et dolore magna
                      aliqua.
                    </v-card-text>
                    <v-card-actions>
                      <v-spacer></v-spacer>
                      <v-btn
                        text="Eu concordo"
                        @click="isActive.value = false"
                      ></v-btn>
                    </v-card-actions>
                  </v-card>
                </template>
              </v-dialog>
            </div>

            <!-- Botões de ação -->
            <div class="d-flex ga-3">
              <v-btn
                type="submit"
                color="success"
                :loading="loading"
                @click="load"
                variant="elevated"
                size="large"
              >
                Enviar denúncia
              </v-btn>

              <v-btn
                @click="handleReset"
                color="error"
                size="large"
                variant="elevated"
              >
                Limpar formulário
              </v-btn>
            </div>
          </div>
        </form>
      </v-card>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useField, useForm } from "vee-validate";
import "leaflet/dist/leaflet.css";
import L from "leaflet";

// Configuração do formulário e campos com validação
const { handleSubmit, handleReset } = useForm({
  validationSchema: {
    name(value) {
      if (value?.length >= 3) return true;
      return "O título deve ter no mínimo 3 caracteres.";
    },
    denuncia(value) {
      if (value?.length >= 7) return true;
      return "A descrição deve ter no mínimo 7 caracteres.";
    },
    localDenuncia(value) {
      if (value) return true;
      return "O campo local é obrigatório.";
    },
    select(value) {
      if (value) return true;
      return "Selecione uma categoria.";
    },
    checkbox(value) {
      if (value === "1") return true;
      return "Você precisa concordar com os termos.";
    },
  },
});

// Definição dos campos do formulário
const name = useField("name");
const denuncia = useField("denuncia");
const localDenuncia = useField("localDenuncia");
const select = useField("select");
const checkbox = useField("checkbox");
const loading = ref(false);
const loadingLocation = ref(false);

// Lista de categorias disponíveis
const items = ref([
  "Segurança Pública",
  "Transporte e Trânsito",
  "Saúde Pública",
  "Maus-Tratos a Animais",
  "Meio Ambiente",
  "Infraestrutura e Serviços Públicos",
  "Corrupção e Fraudes",
  "Direitos Humanos e Sociais",
  "Educação",
  "Habitação",
  "Outro",
]);

// Estado dos arquivos enviados
const files = ref([]);

// Variáveis do mapa
const map = ref(null);
const searchQuery = ref("");
const currentMarker = ref(null);

// Inicialização do mapa
onMounted(() => {
  // Corrige o problema do ícone do marcador
  delete L.Icon.Default.prototype._getIconUrl;
  L.Icon.Default.mergeOptions({
    iconRetinaUrl:
      "https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png",
    iconUrl:
      "https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png",
    shadowUrl:
      "https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png",
  });

  // Inicializa o mapa centralizando no Brasil
  map.value = L.map("map").setView([-15.7801, -47.9292], 4);

  // Adiciona camada do OpenStreetMap
  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "© OpenStreetMap contributors",
  }).addTo(map.value);

  // Adiciona evento de clique no mapa
  map.value.on("click", async (e) => {
    const { lat, lng } = e.latlng;
    await reverseGeocode(lat, lng);
  });
});

// Função para obter localização atual do usuário
const getCurrentLocation = () => {
  loadingLocation.value = true;

  if (!navigator.geolocation) {
    alert("Geolocalização não é suportada pelo seu navegador");
    loadingLocation.value = false;
    return;
  }

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const { latitude, longitude } = position.coords;
      await reverseGeocode(latitude, longitude);
      loadingLocation.value = false;
    },
    (error) => {
      console.error("Erro ao obter localização:", error);
      alert(
        "Não foi possível obter sua localização. Por favor, verifique as permissões do navegador."
      );
      loadingLocation.value = false;
    },
    {
      enableHighAccuracy: true,
      timeout: 5000,
      maximumAge: 0,
    }
  );
};

// Função para buscar endereço
const searchLocation = async () => {
  try {
    const response = await fetch(
      `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(
        searchQuery.value
      )}&limit=1`
    );
    const data = await response.json();

    if (data && data.length > 0) {
      const { lat, lon, display_name } = data[0];
      updateMarkerAndAddress(parseFloat(lat), parseFloat(lon), display_name);
    }
  } catch (error) {
    console.error("Erro ao buscar localização:", error);
  }
};

// Função para geocodificação reversa
const reverseGeocode = async (lat, lng) => {
  try {
    const response = await fetch(
      `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}`
    );
    const data = await response.json();

    if (data && data.display_name) {
      updateMarkerAndAddress(lat, lng, data.display_name);
    }
  } catch (error) {
    console.error("Erro na geocodificação reversa:", error);
  }
};

// Função para atualizar marcador e endereço
const updateMarkerAndAddress = (lat, lng, address) => {
  // Remove marcador anterior se existir
  if (currentMarker.value) {
    map.value.removeLayer(currentMarker.value);
  }

  // Adiciona novo marcador
  currentMarker.value = L.marker([lat, lng]).addTo(map.value);

  // Atualiza o campo de endereço
  localDenuncia.value.value = address;

  // Centraliza o mapa na localização
  map.value.setView([lat, lng], 16);
};

// Função para carregar
const load = () => {
  loading.value = true;
  setTimeout(() => {
    loading.value = false;
  }, 2000);
};

// Função para envio do formulário
const submit = handleSubmit((values) => {
  alert(JSON.stringify(values, null, 2));
});
</script>

<style scoped>
.form-container {
  min-height: 100vh;
  /* background-color: #f5f5f5; */
  padding: 2rem 1rem;
}

.content-wrapper {
  max-width: 1200px;
  margin: 0 auto;
}

.map-container {
  height: 400px;
  width: 100%;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
  overflow: hidden;
}

.form-grid {
  display: grid;
  gap: 1rem;
}

.terms-link {
  color: white;
  text-decoration: underline;
  cursor: pointer;
  margin-left: 4px;
  margin-top: -22px;
}

@media (max-width: 600px) {
  .form-container {
    padding: 1rem;
  }
}
</style>
