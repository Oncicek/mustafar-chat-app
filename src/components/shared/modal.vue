<template>
  <div class="modal-mask" :class="{ 'get-black': props.backColor > 0 }">
    <div class="modal-wrapper">
      <div class="modal-container">
        <div class="modal-header">
          <slot name="header"> default header </slot>
        </div>
        <div class="modal-body">
          <slot name="body"> </slot>
        </div>
        <div class="modal-footer">
          <slot name="footer"> </slot>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { onMounted } from 'vue'

export default {
  props: {
    backColor: Number,
  },

  setup(props: any) {
    onMounted(() => {})

    return {
      props,
    }
  },
}
</script>

<style lang="scss" scoped>
.get-black {
  background-color: black !important;
}

.modal-mask {
  position: fixed;
  z-index: 9998;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: table;
}

.modal-wrapper {
  display: table-cell;
  vertical-align: middle;
}

.modal-container {
  width: 300px;
  margin: 0px auto;
  padding: 10px 20px;
  background-color: #fff;
  border-radius: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.33);
  font-family: Helvetica, Arial, sans-serif;
}

.modal-header h3 {
  margin-top: 0;
  color: #42b983;
}

.modal-body {
  margin: 20px 0;
}

.modal-default-button {
  display: block;
  margin-top: 1rem;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.5s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
