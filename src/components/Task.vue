<template>
  <div>
    <p>{{ task.id }} - {{ status }}</p>
    <button @click="$emit('cancel')">Cancel</button>
  </div>
</template>

<script>
export default {
  props: {
    task: Object,
    status: String,
  },
};
</script>
