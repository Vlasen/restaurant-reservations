<script setup>
import Calendar from './Calendar.vue'
import IconZoomeOut from './icons/IconZoomeOut.vue'
import IconZoomeIn from './icons/IconZoomeIn.vue'
import IconLoading from './icons/IconLoading.vue'
import { parseISO, format, addDays } from 'date-fns';
import { ru } from 'date-fns/locale';
import { formatInTimeZone } from 'date-fns-tz';
import { ref, reactive, onMounted, computed  } from 'vue'
import axios from 'axios'

const emit = defineEmits(['restaurant-loaded'])

const restaurantData = reactive({
  restaurant: {
    id: '',
    opening_time: '',
    closing_time: '',
    restaurant_name: '',
    timezone: ''
  },
  tables: [],
});
const originalTables = ref([]);
const days = ref([]);
const zones = [
  '1 Этаж', 
  '2 Этаж', 
  'Банкетный зал'
];
const currentDate = ref('');
const availableDays = ref([]);
const selectedDate = ref('');
const selectedZones = ref([...zones]);
const timeList = ref([]);
const isLoading =ref(false);

async function getRestaurantData() {
  isLoading.value = true;
  try {
    const { data } = await axios.get('https://hh.frontend.ark.software/api/booking');

    if (data?.restaurant) {
      Object.assign(restaurantData.restaurant, data.restaurant)
    } else {
      console.error('Ресторан не найден:', data)
    }
    
    originalTables.value = (data.tables || []).map(table => ({
      ...table,
      orders: Array.isArray(table.orders) ? table.orders : [],
      reservations: Array.isArray(table.reservations) ? table.reservations : []
    }));
    restaurantData.tables = [...originalTables.value];

    currentDate.value = data.current_day;
    availableDays.value = data.available_days;
    days.value = availableDays.value.map(dateStr =>
      formatAvailableDay(dateStr, restaurantData.restaurant.timezone || 'Europe/Moscow')
    );
    changeValuesDays(days.value);
  } catch (error) {
    console.error('Ошибка при загрузке данных:', error);
  } finally {
    isLoading.value = false;
  }
};

function formatAvailableDay(dateStr, timeZone) {
  const date = parseISO(dateStr);

  const formattedDate = formatInTimeZone(date, timeZone, 'd MMMM', { locale: ru });
  const weekday = formatInTimeZone(date, timeZone, 'EEEE', { locale: ru });

  return {
    original: dateStr,
    date: formattedDate,
    weekday: weekday
  };
}
function changeValuesDays (daysArray) {
  const todayISO = currentDate.value;
  const tomorrowISO = format(addDays(parseISO(todayISO), 1), 'yyyy-MM-dd');

  daysArray.forEach(day => {
    if (day.original === currentDate.value) {
      day.weekday = 'сегодня';
    }
    if (day.original === tomorrowISO) {
      day.weekday = 'завтра';
    }
  });
}

function selectDate(dateStr) {
  selectedDate.value = dateStr;
  updateFilteredTables();
}

function toggleZone(zone) {
  const index = selectedZones.value.indexOf(zone);
  if (index >= 0) {
    selectedZones.value.splice(index, 1);
  } else {
    selectedZones.value.push(zone);
  }
  updateFilteredTables();
}

function updateFilteredTables() {
  const dateStr = selectedDate.value;
  const filteredTables = originalTables.value.map(table => {
    const filteredOrders = table.orders.filter(order => {
      const orderDate = order.start_time.split('T')[0];
      return orderDate === dateStr;
    });
    const filteredReservations = table.reservations.filter(res => {
      const seatingDate = res.seating_time?.split('T')[0];
      return seatingDate === dateStr;
    });

    if (!selectedZones.value.includes(table.zone)) {
      return null;
    }

    return {
      ...table,
      orders: filteredOrders,
      reservations: filteredReservations
    };
  }).filter(Boolean);
  
  restaurantData.tables = filteredTables;
  generateTimeSlots(
    restaurantData.restaurant.opening_time,
    restaurantData.restaurant.closing_time
  );
}

function generateTimeSlots(openingTime, closingTime) {
  timeList.value = []
  const [openHours, openMinutes] = openingTime.split(':').map(Number);
  const [closeHours, closeMinutes] = closingTime.split(':').map(Number);

  const openingInMinutes = openHours * 60 + openMinutes;
  const closingInMinutes = closeHours * 60 + closeMinutes;

  let current = openingInMinutes;

  while (current < closingInMinutes) {
    const hours = Math.floor(current / 60);
    const minutes = current % 60;
    timeList.value.push(
      `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`
    );
    current += 30;
  }
}

const baseCellWidth = 80;
const baseCellHeight = 40;
const currentCellWidth = ref(baseCellWidth);
const currentCellHeight = ref(baseCellHeight);
const horizontalStep = 16;
const verticalStep = 4;
const scaleX = computed(() => currentCellWidth.value / baseCellWidth);
const scaleY = computed(() => currentCellHeight.value / baseCellHeight);
function zoomIn() {
  currentCellWidth.value += horizontalStep;
  currentCellHeight.value += verticalStep;
}
function zoomOut() {
  if (currentCellWidth.value > horizontalStep && currentCellHeight.value > verticalStep) {
    currentCellWidth.value -= horizontalStep;
    currentCellHeight.value -= verticalStep;
  }
}

onMounted(async () => {
  await getRestaurantData();
  emit('restaurant-loaded', restaurantData.restaurant.restaurant_name);
  selectedDate.value = currentDate.value;
  updateFilteredTables();
});
</script>

<template>
  <div class="circle-loading" v-if="isLoading">
    <IconLoading/>
  </div>

  <label class="h1">Бронирования</label>
  <div class="filter-wrapper" >
    <div class="date-filter">
      <label for="choose-date-btn" class="h2">Дата</label>
      <div id="choose-date-btn" class="buttons-container">
        <button 
          v-for="day in days" 
          :class="['filter-btn', day.original === selectedDate ? 'selected' : '']"
          type="button"
          @click="selectDate(day.original)"
        >
          <span class="h2-bold">{{ day.date }}</span>
          <span class="h2">{{ day.weekday }}</span>
        </button>
      </div>
    </div>

    <div class="zone-filter">
      <label for="submit-btn" class="h2">Отображаемые зоны</label>
      <div 
        id="choose-zone-btn" 
        class="buttons-container"
      >
        <button 
          v-for="zone in zones" 
          :class="['filter-btn', selectedZones.includes(zone) ? 'selected' : '']"
          type="button"
          @click="toggleZone(zone)"
        >
          <span class="h2">{{ zone }}</span>
        </button>
      </div>
    </div>

  </div>
  <div class="table-container" 
    :style="{
      transform: `scale(${scaleX}, ${scaleY})`,
      transformOrigin: 'top left'
    }">
    <Calendar
      :timeList="timeList"
      :restaurantData="restaurantData"
    />
  </div>
  
  <div class="scale-wrapper">
    <span class="text">Масштаб</span>
    <div class="buttons-row">
      <button class="zoom-btn" type="button" @click="zoomOut">
        <IconZoomeOut/>
      </button>
      <button class="zoom-btn" type="button" @click="zoomIn">
        <IconZoomeIn/>
      </button>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.circle-loading {
  position: fixed;
  background: var(--bg-color);
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  opacity: .7;
  z-index: 100;
}
.filter-wrapper {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 33px;

  @mixin filter-container {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .date-filter {
    @include filter-container;
  }
  .zone-filter {
    @include filter-container;
  }
  .buttons-container {
    display: flex;
    flex-direction: row;
    gap: 8px;
    .filter-btn {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: flex-start;
      padding: 4px 8px;
      width: auto;
      height: auto;
      min-height: 36px;
      background: var(--bg-color-light);
      border-radius: 8px;

      span {
        color: var(--text-color);
      }
    }
    .filter-btn.selected {
      background: var(--selected-btn-color);
      span {
        color: white;
      }
    }
    .filter-btn.selected:hover {
      background: var(--selected-btn-hover-color);
    }
    .filter-btn:hover {
      background: var(--btn-color-hover);
    }
  }
  #choose-zone-btn {
    .filter-btn {
      min-height: 24px;
      border-radius: 4px;
      padding: 0 6px;
      
      span {
        text-transform: lowercase;
      }
      span::first-letter {
        text-transform: uppercase;
      }
    }
  }
}
.scale-wrapper {
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  gap: 4px;
  position: fixed;
  bottom: 30px;
  right: 30px;
  width: 89px;
  height: 60px;
  padding: 8px;
  background: var(--bg-color);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  z-index: 20;

  .buttons-row {
    display: flex;
    flex-direction: row;
    height: auto;
    gap: 4px;

    .zoom-btn {
      width: 24px;
      height: 24px;
      background-color: var(--btn-color);
      border-radius: 4px;
    }
    .zoom-btn:hover {
      background: var(--btn-color-hover);
    }
  }
}
</style>
