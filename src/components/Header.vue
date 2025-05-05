<script setup>
import IconTheme from './icons/IconTheme.vue'
import IconLogout from './icons/IconLogout.vue'
import { ref, onMounted } from 'vue'

defineProps({
  restaurant: String
})

const isDark = ref(false);

function toggleTheme() {
  isDark.value = !isDark.value;
  const html = document.documentElement;
  if (isDark.value) {
    html.classList.add('dark');
    localStorage.setItem('theme', 'dark');
  } else {
    html.classList.remove('dark');
    localStorage.setItem('theme', 'light');
  }
}

onMounted(() => {
  const savedTheme = localStorage.getItem('theme');

  if (savedTheme === 'dark') {
    isDark.value = true;
    document.documentElement.classList.add('dark');
  } else {
    isDark.value = false;
    document.documentElement.classList.remove('dark');
  }
});
</script>

<template>
  <div class="wrapper">
    <div class="label h2-bold">
      <span>AIRESTO</span>
      <span v-if="restaurant">|</span>
      <span>{{ restaurant }}</span>
    </div>
    <div class="search-and-buttons">
      <input placeholder="⌘+Л поиск по имени"/>
      <div class="icon"></div>
      <button type="button" class="btn-theme" @click="toggleTheme">
        <IconTheme/>
      </button>
      <button type="button" class="btn-logout">
        <IconLogout/>
        <span class="h2">Выйти</span>
      </button>
    </div>
  </div>
</template>

<style scoped lang="scss">
.wrapper {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  padding: 0px 20px;
  gap: 10px;
  max-width: 100%;
  height: 44px;
  background: var(--header-color);
  .label {
    display: flex;
    flex-wrap: nowrap;
    gap: 4px;
  }
  .search-and-buttons {
    position: relative;
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 8px;

    input {
      box-sizing: border-box;
      display: flex;
      flex-direction: row;
      align-items: center;
      padding: 0px 8px 0 36px;
      gap: 8px;
      width: 258px;
      height: 28px;
      background: var(--input-color);
      border: 1px solid var(--border-color);
      border-radius: 8px;
      color: var(--text-color);
    }
    input:focus {
      outline: none;
      border: 1px solid var(--input-focus-color);
    }
    .icon {
      position: absolute;
      top: 2px;
      left: 8px;
      mask-image: url('../assets/search.svg');
      mask-repeat: no-repeat;
      mask-position: center;
      mask-size: contain;
      background-color: var(--search-icon-color);
      width: 24px;
      height: 24px;
    }
  
    .btn-theme {
      width: 24px;
      height: 24px;
      padding: 0;
    }
    .button-theme:hover {
      opacity: 0.8;
    }

    .btn-logout {
      flex-direction: row;
      width: 66px;
      height: 24px;
      gap: 4px;

      span {
        line-height: 16px;
        color: white;
      }
    }
  }
}
</style>