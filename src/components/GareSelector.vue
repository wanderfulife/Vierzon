<template>
  <div class="form-group">
    <input
      class="form-control"
      type="text"
      v-model="searchTerm"
      placeholder="Nom Gare"
      @focus="showDropdown = true"
    />
    <ul
      v-if="showDropdown && searchTerm && filteredGares.length"
      class="gare-dropdown"
    >
      <li
        v-for="item in filteredGares"
        :key="item['Code UIC']"
        @click="selectGare(item)"
      >
        {{ item["Nom Gare"] }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, watch, computed } from "vue";
import gares from "./gare.json";

const showDropdown = ref(false);

const props = defineProps({
  modelValue: String,
});
const emit = defineEmits(["update:modelValue"]);

const searchTerm = ref(props.modelValue || "");

watch(
  () => props.modelValue,
  (newValue) => {
    searchTerm.value = newValue || "";
  }
);

const filteredGares = computed(() => {
  if (!searchTerm.value) return [];
  const searchLower = searchTerm.value.toLowerCase();
  const uniqueGares = new Map();

  for (const item of gares) {
    const nomGare = item["Nom Gare"]?.toLowerCase() ?? "";
    if (nomGare.includes(searchLower)) {
      uniqueGares.set(item["Nom Gare"], item);
    }
  }

  return Array.from(uniqueGares.values());
});

const selectGare = (gare) => {
  searchTerm.value = gare["Nom Gare"];
  showDropdown.value = false;
  emit("update:modelValue", gare["Nom Gare"]);
};
</script>

<style>
.form-group {
  width: 100%;
  position: relative;
}

.form-control {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 16px;
  background-color: white;
  color: #333;
}

.gare-dropdown {
  position: absolute;
  width: 100%;
  list-style-type: none;
  padding: 0;
  margin: 0;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  max-height: 200px;
  overflow-y: auto;
  background-color: white;
  z-index: 1000;
}

.gare-dropdown li {
  padding: 10px;
  cursor: pointer;
  color: #333;
  background-color: white;
  border-bottom: 1px solid #eee;
}

.gare-dropdown li:hover {
  background-color: #f5f5f5;
}

.gare-dropdown li:last-child {
  border-bottom: none;
}
</style>