<template>
  <div class="viewer-container">
    <div id="viewer" ref="viewerRef"></div>
    <div class="controls">
      <h1>Hotspot Viewer</h1>
      <!-- <button @click="zoomToCenter">Zoom to Center</button>
      <button @click="zoomToLevel(2)">Zoom x2</button>
      <button @click="smoothZoom">Smooth Zoom</button>
      <button @click="zoomToAllHotspots">Fit All Hotspots</button> -->
    </div>
    <transition name="sidebar">
      <div class="sidebar" v-if="showSidebar">
        <div class="sidebar-content">
          <button @click="closeSidebar" class="close-btn">
            <div>×</div>
          </button>
          <h2>{{ selectedHotspot?.title }}</h2>
          <p>{{ selectedHotspot?.content }}</p>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import OpenSeadragon from "openseadragon";

// Reactive variables
const viewer = ref(null);
const showSidebar = ref(false);
const selectedHotspot = ref(null);
const viewerRef = ref(null);
const imageDimensions = ref({ width: 0, height: 0 });
const trackers = ref([]);

// Hotspots with relative sizes (percentages of image dimensions)
const hotspotBase = [
  {
    id: 1,
    x: 450, // pixel
    y: 180, // pixel
    size: 0.05, // 5% of smallest dimension
    title: "Discovery Zone",
    content:
      "Discover the captivating charm of this historic area, full of rich details and fascinating stories. With this versatile tripod featuring a head, you'll have all you need to take professional-quality photos. The advanced 90° center column allows you to extend the tripod vertically or horizontally, letting you frame and shoot within seconds. Effortlessly secure your setup with the quick power lock system, and easily fold the tripod into a compact, portable size when not in use. You can even reposition the tripod with your camera still attached, ensuring you capture the perfect shot every time.📸✨",
  },
  {
    id: 2,
    x: 880, // pixel
    y: 420, // pixel
    size: 0.05, // 5% of smallest dimension
    title: "Hidden Gem",
    content:
      "Uncover a hidden gem brimming with incredible features just waiting to be discovered. This secret spot offers breathtaking views, unique landscapes, and captivating history that will leave you in awe. From tranquil hiking trails to scenic viewpoints, every corner has something special to offer. Whether you're an adventure seeker or simply looking for a peaceful retreat, this enchanting location promises an unforgettable experience. 🌿✨",
  },
  {
    id: 3,
    x: 1360,
    y: 330,
    size: 0.05, // 5% of smallest dimension
    title: "Hidden Gem (2)",
    content:
      "Explore an undiscovered paradise filled with hidden wonders awaiting your arrival. 🌄🌿✨",
  },
];

const initViewer = () => {
  if (!viewerRef.value) {
    console.error("Viewer element not found");
    return;
  }

  viewer.value = OpenSeadragon({
    element: viewerRef.value,
    prefixUrl: "https://openseadragon.github.io/openseadragon/images/",
    tileSources: {
      type: "image",
      // url: "https://picsum.photos/3000/1500", // Your image URL here
      url: "https://images.unsplash.com/photo-1741377772075-5f0f0d21d6b4?q=80&w=2000&h=2000",
    },
    showNavigationControl: false,
    gestureSettingsMouse: {
      scrollToZoom: false,
      clickToZoom: false,
      dblClickToZoom: false,
      pinchToZoom: false,
      flickEnabled: false,
    },
    gestureSettingsTouch: {
      pinchZoom: false,
      flickEnabled: false,
      dblClickToZoom: false,
    },
    defaultZoomLevel: 1,
    minZoomLevel: 1,
    maxZoomLevel: 3,
    zoomPerClick: 1,
    animationTime: 1.5,
    degrees: 0,
  });

  viewer.value.addHandler("open", () => {
    const tiledImage = viewer.value.world.getItemAt(0);
    imageDimensions.value = {
      width: tiledImage.getContentSize().x,
      height: tiledImage.getContentSize().y,
    };
    console.log("Image dimensions:", imageDimensions.value);
    addHotspots();
  });

  viewer.value.addHandler("open-failed", () => {
    console.error("Failed to load image");
    alert("Failed to load image");
  });
};

// Convert relative coordinates and sizes to pixels
const getHotspotPixels = (hotspot) => {
  const imgWidth = imageDimensions.value.width;
  const imgHeight = imageDimensions.value.height;

  return {
    ...hotspot,
    x: hotspot.x,
    y: hotspot.y,
    width: hotspot.size * imgWidth,
    height: hotspot.size * imgHeight,
  };
};

const pixelsToImageCoords = (x, y, width, height) => {
  if (!imageDimensions.value.width || !imageDimensions.value.height) {
    console.error("Image dimensions not available");
    return new OpenSeadragon.Rect(0, 0, 0, 0);
  }

  const imgWidth = imageDimensions.value.width;
  const imgHeight = imageDimensions.value.height;
  const rect = new OpenSeadragon.Rect(
    x / imgWidth,
    y / imgHeight,
    width / imgWidth,
    height / imgHeight
  );
  return rect;
};

const addHotspots = () => {
  if (!viewer.value) {
    console.error("Viewer not initialized");
    return;
  }

  hotspotBase.forEach((baseHotspot) => {
    const hotspot = getHotspotPixels(baseHotspot);
    const overlay = document.createElement("div");
    overlay.className = "hotspot";
    overlay.dataset.id = hotspot.id;
    overlay.style.zIndex = "100";

    const rect = pixelsToImageCoords(
      hotspot.x,
      hotspot.y,
      hotspot.width,
      hotspot.height
    );
    console.log("Adding overlay for hotspot", hotspot.id, "at", rect);

    viewer.value.addOverlay({
      element: overlay,
      location: rect,
      checkResize: false,
    });

    const tracker = new OpenSeadragon.MouseTracker({
      element: overlay,
      clickHandler: (event) => {
        event.preventDefaultAction = true;
        console.log(
          "Hotspot clicked via MouseTracker:",
          hotspot.id,
          hotspot.title
        );
        handleHotspotClick(hotspot);
      },
      pressHandler: () => console.log("Hotspot pressed:", hotspot.id),
      releaseHandler: () => console.log("Hotspot released:", hotspot.id),
    });

    trackers.value.push(tracker);
    console.log("MouseTracker attached to hotspot", hotspot.id);
  });
};

const handleHotspotClick = (hotspot) => {
  console.log("Handling hotspot click:", hotspot.title);
  selectedHotspot.value = hotspot;
  showSidebar.value = true;
  zoomToHotspot(hotspot);
};

const zoomToHotspot = (hotspot) => {
  const padding = imageDimensions.value.width * 0.05;
  const rect = pixelsToImageCoords(
    hotspot.x - padding,
    hotspot.y - padding,
    hotspot.width + padding * 2,
    hotspot.height + padding * 2
  );
  console.log("Zooming to hotspot:", rect);
  viewer.value.viewport.fitBounds(rect);
};

const zoomToCenter = () => {
  const center = viewer.value.viewport.getCenter();
  viewer.value.viewport.zoomTo(2, center);
};

const zoomToLevel = (level) => {
  const center = viewer.value.viewport.getCenter();
  viewer.value.viewport.zoomTo(level, center);
};

const smoothZoom = async () => {
  const center = viewer.value.viewport.getCenter();
  const steps = [1.5, 2, 2.5, 2, 1.5, 1];

  for (const level of steps) {
    viewer.value.viewport.zoomTo(level, center, true);
    await new Promise((resolve) => setTimeout(resolve, 600));
  }
};

const zoomToAllHotspots = () => {
  if (hotspotBase.length === 0) return;

  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;
  const padding = imageDimensions.value.width * 0.1;

  hotspotBase.forEach((baseHotspot) => {
    const hotspot = getHotspotPixels(baseHotspot);
    minX = Math.min(minX, hotspot.x - padding);
    minY = Math.min(minY, hotspot.y - padding);
    maxX = Math.max(maxX, hotspot.x + hotspot.width + padding);
    maxY = Math.max(maxY, hotspot.y + hotspot.height + padding);
  });

  const rect = pixelsToImageCoords(minX, minY, maxX - minX, maxY - minY);
  viewer.value.viewport.fitBounds(rect);
};

const closeSidebar = () => {
  showSidebar.value = false;
  selectedHotspot.value = null;

  // wait for sidebar to close before zooming out
  setTimeout(() => {
    viewer.value.viewport.goHome();
  }, 500);
};

const preventZoom = (e) => {
  if (e.ctrlKey || e.metaKey) {
    e.preventDefault();
  }
};

onMounted(() => {
  initViewer();
  if (viewerRef.value) {
    viewerRef.value.addEventListener("wheel", preventZoom, { passive: false });
  } else {
    console.error(
      "Failed to add wheel event listener: viewer element not found"
    );
  }
});
</script>

<style scoped>
.viewer-container {
  display: flex;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  background: #1a1a1a;
  position: relative;
}

#viewer {
  flex: 1;
  height: 100%;
  transition: all 0.3s ease;
}

.sidebar {
  position: absolute;
  width: 350px;
  right: 0;
  height: 100%;
  background: linear-gradient(135deg, #ffffff, #f0f0f0);
  box-shadow: -5px 0 15px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
  z-index: 9999;
}

.sidebar-content {
  padding: 2rem;
}

.controls {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 100;
  display: flex;
  gap: 10px;
}

.controls h1 {
  padding: 10px;
  font-size: 1.8rem;
  color: white;
  background-color: rgba(0, 0, 0, 0.5);
}

.controls button {
  padding: 8px 16px;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.controls button:hover {
  background: #ff8787;
  transform: translateY(-2px);
}

.sidebar h2 {
  color: #2c3e50;
  margin-top: 0;
  font-size: 1.8rem;
}

.sidebar p {
  color: #666;
  line-height: 1.6;
}

.close-btn {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  width: 2.2rem;
  height: 2.2rem;
  border: none;
  background: #ff6b6b;
  color: white;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 1.4rem;
  box-sizing: border-box;
  padding: 0px;
}

.close-btn:hover {
  background: #ff8787;
  transform: rotate(90deg);
}

.sidebar-enter-active,
.sidebar-leave-active {
  transition: right 0.4s ease;
}

.sidebar-enter-from,
.sidebar-leave-to {
  /* sidebar width */
  right: -350px;
}

@media (max-width: 768px) {
  .sidebar {
    width: 100%;
    position: absolute;
    right: 0;
    height: 30dvh;
    top: 70dvh;
  }
  .controls {
    flex-direction: column;
    width: 250px;
  }
  .close-btn {
    position: fixed;
    top: auto;
    bottom: 27.5dvh;
  }
  .sidebar-enter-active,
  .sidebar-leave-active {
    transition: top 0.4s ease;
  }
  .sidebar-enter-from,
  .sidebar-leave-to {
    top: 100dvh;
  }
  .sidebar-enter-active .close-btn,
  .sidebar-leave-active .close-btn {
    transition: bottom 0.4s ease;
  }
  .sidebar-enter-from .close-btn,
  .sidebar-leave-to .close-btn {
    /* button height */
    bottom: -2.2rem;
  }
}
</style>
