<template>
  <div
    v-if="visible"
    class="context-menu"
    :style="{ top: `${y}px`, left: `${x}px` }"
    @click.stop
  >
    <ul>
      <li v-for="item in items" :key="item.label" @click="onItemClick(item)">
        {{ item.label }}
      </li>
    </ul>
  </div>
</template>

<script setup>

const props = defineProps({
  visible: Boolean,
  x: Number,
  y: Number,
  items: {
    type: Array,
    default: () => [] // Example: [{ label: 'Verwijderen', action: 'delete' }]
  }
});

const emit = defineEmits(['item-click']);

const onItemClick = (item) => {
  emit('item-click', item.action);
};
</script>

<style scoped>
.context-menu {
  position: absolute;
  background-color: white;
  border: 1px solid #ccc;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.15);
  min-width: 100px;
  z-index: 1000; /* Ensure it's above other elements */
}

.context-menu ul {
  list-style: none;
  padding: 5px 0;
  margin: 0;
}

.context-menu li {
  padding: 8px 12px;
  cursor: pointer;
}

.context-menu li:hover {
  background-color: #eee;
}
</style>