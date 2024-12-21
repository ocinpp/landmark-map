<template>
  <div class="h-screen flex flex-col">
    <div class="h-16 bg-gray-800 flex items-center justify-center">
      <h1 class="text-white text-2xl font-bold">Tokyo Landmarks</h1>
    </div>
    <div class="flex-grow relative flex overflow-hidden">
      <!-- Sidebar container -->
      <div
        :class="[
          'transition-all duration-300 ease-in-out z-[1100] flex h-full',
          isSidebarOpen ? 'w-64' : 'w-8',
        ]"
      >
        <!-- Sidebar content -->
        <div
          :class="[
            'h-full bg-white shadow-lg overflow-y-auto flex-shrink-0',
            isSidebarOpen ? 'w-64' : 'w-8',
          ]"
        >
          <div v-if="isSidebarOpen">
            <div
              v-for="(group, groupName) in groupedLocations"
              :key="groupName"
              class="mb-2"
            >
              <button
                @click="toggleGroup(groupName)"
                class="w-full px-4 py-2 text-left font-semibold bg-gray-100 hover:bg-gray-200 focus:outline-none"
                :aria-expanded="expandedGroups[groupName]"
              >
                {{ groupName }}
              </button>
              <div v-show="expandedGroups[groupName]">
                <button
                  v-for="location in group"
                  :key="location.name"
                  @click="panToLocation(location)"
                  class="w-full px-6 py-2 text-left hover:bg-gray-100 focus:outline-none"
                >
                  {{ location.name }}
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Sidebar toggle button -->
        <button
          @click="toggleSidebar"
          :class="[
            'absolute top-2 z-[1101] bg-white p-2 rounded-md shadow-md',
            isSidebarOpen ? 'left-[248px]' : 'left-2',
          ]"
          aria-label="Toggle sidebar"
        >
          <MenuIcon v-if="!isSidebarOpen" class="h-6 w-6" />
          <XIcon v-else class="h-6 w-6" />
        </button>
      </div>

      <!-- Map container -->
      <div class="flex-grow relative overflow-hidden">
        <div id="map" class="absolute inset-0"></div>

        <!-- Location Details Overlay -->
        <transition
          enter-active-class="transition-transform duration-300 ease-out"
          leave-active-class="transition-transform duration-300 ease-in"
          enter-from-class="translate-y-full sm:translate-x-full"
          enter-to-class="translate-y-0 sm:translate-x-0"
          leave-from-class="translate-y-0 sm:translate-x-0"
          leave-to-class="translate-y-full sm:translate-x-full"
        >
          <div
            v-if="selectedLocation"
            class="absolute right-0 w-full sm:w-96 bg-white shadow-lg overflow-y-auto transform sm:translate-y-0 z-[1100] sm:top-0 sm:bottom-0 bottom-0 max-h-[50vh] sm:max-h-full flex flex-col"
          >
            <div class="p-6 bg-white sticky top-0 z-10 border-b">
              <div class="flex justify-between items-center">
                <h2 class="text-2xl font-bold">{{ selectedLocation.name }}</h2>
                <button
                  @click="closeOverlay"
                  class="text-gray-500 hover:text-gray-700"
                  aria-label="Close details"
                >
                  <XIcon class="h-6 w-6" />
                </button>
              </div>
            </div>
            <div class="p-6 overflow-y-auto flex-grow">
              <img
                :src="selectedLocation.image"
                :alt="selectedLocation.name"
                class="w-full object-cover rounded-lg mb-4"
              />
              <p class="text-gray-600 mb-4">
                {{ selectedLocation.description }}
              </p>
              <a
                :href="selectedLocation.website"
                target="_blank"
                rel="noopener noreferrer"
                class="inline-flex items-center px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition duration-300"
              >
                <GlobeIcon class="h-5 w-5 mr-2" />
                Visit Official Website
              </a>
            </div>
          </div>
        </transition>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, computed } from "vue";
import "leaflet/dist/leaflet.css";
import L from "leaflet";
import { XIcon, GlobeIcon, MenuIcon } from "lucide-vue-next";

const selectedLocation = ref(null);
const map = ref(null);
const markers = ref({});
const expandedGroups = ref({ Sightseeing: true, Art: true });
const isSidebarOpen = ref(true);

const locations = [
  {
    name: "Tokyo Tower",
    lat: 35.6586,
    lng: 139.7454,
    description:
      "Tokyo Tower is a communications and observation tower in the Shiba-koen district of Minato, Tokyo, Japan. At 332.9 meters (1,092 ft), it is the second-tallest structure in Japan.",
    image:
      "https://images.unsplash.com/photo-1713978504175-f7f4b0aefc3b?h=300&q=100",
    website: "https://www.tokyotower.co.jp/en/",
    group: "Sightseeing",
  },
  {
    name: "Tokyo Skytree",
    lat: 35.71,
    lng: 139.8107,
    description:
      "Tokyo Skytree is a broadcasting and observation tower in Sumida, Tokyo, Japan. It became the tallest structure in Japan in 2010 and reached its full height of 634.0 meters (2,080 ft) in March 2011.",
    image:
      "https://images.unsplash.com/photo-1719675531669-5fc58cb0164c?q=100&h=300",
    website: "http://www.tokyo-skytree.jp/en/",
    group: "Sightseeing",
  },
  {
    name: "Tokyo Disneyland",
    lat: 35.6329,
    lng: 139.8804,
    description:
      "Tokyo Disneyland is a 115-acre theme park at the Tokyo Disney Resort in Urayasu, Chiba Prefecture, Japan, near Tokyo. It was the first Disney park to be built outside the United States and opened on April 15, 1983.",
    image:
      "https://images.unsplash.com/photo-1624601573012-efb68931cc8f?h=300&q=100",
    website: "https://www.tokyodisneyresort.jp/en/tdl/",
    group: "Sightseeing",
  },
  {
    name: "The National Museum of Modern Art Tokyo",
    lat: 35.6905,
    lng: 139.7546,
    description:
      "The National Museum of Modern Art, Tokyo (MOMAT) is Japan's first national art museum. It is known for its collection of 20th-century Japanese art and international art.",
    image:
      "https://upload.wikimedia.org/wikipedia/commons/5/5c/National_Museum_of_Modern_Art%2C_Tokyo.jpg",
    website: "https://www.momat.go.jp/english/",
    group: "Art",
  },
];

const groupedLocations = computed(() => {
  return locations.reduce((acc, location) => {
    if (!acc[location.group]) {
      acc[location.group] = [];
    }
    acc[location.group].push(location);
    return acc;
  }, {});
});

const toggleGroup = (groupName) => {
  expandedGroups.value[groupName] = !expandedGroups.value[groupName];
};

const toggleSidebar = () => {
  isSidebarOpen.value = !isSidebarOpen.value;
};

const panToLocation = (location) => {
  if (map.value) {
    map.value.flyTo(L.latLng(location.lat, location.lng), 14);
    selectedLocation.value = location;
    markers.value[location.name].openPopup();
  }
};

const closeOverlay = () => {
  selectedLocation.value = null;
};

const handleResize = () => {
  if (selectedLocation.value) {
    panToLocation(selectedLocation.value);
  }
};

onMounted(() => {
  map.value = L.map("map", {
    zoomControl: false,
    closePopupOnClick: false,
  }).setView([35.6762, 139.7793], 11);

  L.control
    .zoom({
      position: "bottomleft",
    })
    .addTo(map.value);

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
  }).addTo(map.value);

  locations.forEach((location) => {
    const marker = L.marker([location.lat, location.lng]).addTo(map.value);
    marker.on("click", () => {
      selectedLocation.value = location;
    });

    const popupContent = `
      <div class="text-center">
        <img src="${location.image}" alt="${location.name}" class="w-32 h-32 object-cover rounded-lg mb-2">
        <strong>${location.name}</strong>
      </div>
    `;
    marker.bindPopup(popupContent, {
      maxWidth: 100,
      className: "custom-popup",
      closeOnClick: false,
      autoClose: true,
      closeButton: true,
    });

    markers.value[location.name] = marker;
  });

  window.addEventListener("resize", handleResize);
});

onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
});
</script>

<style>
@import url("https://unpkg.com/leaflet@1.7.1/dist/leaflet.css");

.custom-popup .leaflet-popup-content-wrapper {
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 12px;
  padding: 10px;
}

.custom-popup .leaflet-popup-content {
  margin: 0;
}

.custom-popup .leaflet-popup-tip-container {
  visibility: hidden;
}
</style>

<style scoped>
#map {
  height: 100%;
  width: 100%;
}
</style>
