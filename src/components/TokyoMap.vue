<template>
  <div class="h-screen flex flex-col">
    <div class="h-16 bg-gray-800 flex items-center justify-center">
      <h1 class="text-white text-2xl font-bold">Tokyo Landmarks</h1>
    </div>
    <div class="flex-grow relative">
      <div id="map" class="h-full w-full"></div>
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
            <p class="text-gray-600 mb-4">{{ selectedLocation.description }}</p>
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
</template>

<script setup>
import { onMounted, onUnmounted, ref, watch } from "vue";
import "leaflet/dist/leaflet.css";
import L from "leaflet";
import { XIcon, GlobeIcon } from "lucide-vue-next";

const selectedLocation = ref(null);
const map = ref(null);
const markers = ref([]);

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
  },
];

watch(selectedLocation, (newValue) => {
  if (newValue && map.value) {
    const windowWidth = window.innerWidth;
    const isMobile = windowWidth < 640; // Assuming 640px as the breakpoint for small screens

    if (isMobile) {
      const mobileOverlayHeight = window.innerHeight * 0.5; // 50% of viewport height
      const visibleMapHeight = window.innerHeight - mobileOverlayHeight;
      const targetY = visibleMapHeight * 0.5; // Position marker at the middle of the visible map area

      const point = map.value.project(L.latLng(newValue.lat, newValue.lng));
      point.y += targetY;
      const newCenter = map.value.unproject(point);

      map.value.setView(newCenter, map.value.getZoom(), {
        animate: true,
        pan: {
          duration: 0.5,
        },
      });
    } else {
      // Desktop logic remains unchanged
      const overlayWidth = 384; // 96 * 4 = 384px (sm:w-96)
      const point = map.value.project(L.latLng(newValue.lat, newValue.lng));
      point.x += overlayWidth / 2;
      const newCenter = map.value.unproject(point);

      map.value.setView(newCenter, map.value.getZoom(), {
        animate: true,
        pan: {
          duration: 0.5,
        },
      });
    }
  }
});

const closeOverlay = () => {
  selectedLocation.value = null;
};

const handleResize = () => {
  if (selectedLocation.value) {
    // Trigger the watch function to recenter the map
    selectedLocation.value = { ...selectedLocation.value };
  }
};

const togglePopups = (zoomLevel) => {
  if (zoomLevel >= 12) {
    markers.value.forEach((marker) => marker.openPopup());
  } else {
    markers.value.forEach((marker) => marker.closePopup());
  }
};

onMounted(() => {
  map.value = L.map("map", { closePopupOnClick: false }).setView(
    [35.6762, 139.6503],
    11
  );

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
  }).addTo(map.value);

  locations.forEach((location) => {
    const marker = L.marker([location.lat, location.lng]).addTo(map.value);
    marker.on("click", () => {
      selectedLocation.value = location;
    });

    // Add popup with image thumbnail
    const popupContent = `
      <div class="text-center">
        <img src="${location.image}" alt="${location.name}" class="w-32 h-32 object-cover rounded-lg mb-2">
        <strong>${location.name}</strong>
      </div>
    `;
    marker.bindPopup(popupContent, {
      maxWidth: 300,
      className: "custom-popup",
      closeOnClick: false,
      autoClose: true,
      closeButton: true,
    });

    markers.value.push(marker);
  });

  // Add zoom event listener
  // map.value.on("zoomend", () => {
  //   const currentZoom = map.value.getZoom();
  //   togglePopups(currentZoom);
  // });

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
}
</style>
