<script setup>
import IconPhone from './icons/IconPhone.vue'
import { defineProps, reactive, ref } from 'vue';
import { parseISO } from 'date-fns';
import { formatInTimeZone } from 'date-fns-tz';

const props = defineProps({
  timeList: Array,
  restaurantData: Object
});

const orderLabels = reactive({
  new: { name: 'Заказ', status: 'Новый', apiStatus: 'New' },
  bill: { name: 'Заказ', status: 'Пречек', apiStatus: 'Bill' },
  closed: { name: 'Заказ', status: 'Закрытый', apiStatus: 'Closed' },
  banquet: { name: 'Банкет', status: '', apiStatus: 'Banquet' }
});

const reservationLabels = reactive([
  { name: 'New', status: 'Ожидает подтверждения', apiStatus: 'Новая' },
  { name: 'Waiting', status: 'Ожидаем', apiStatus: 'Заявка' },
  { name: 'Closed', status: 'Отменен', apiStatus: 'Закрыт' },
  { name: 'Closed', status: 'Отменен', apiStatus: 'Отменен' },
  { name: 'Open', status: 'В зале', apiStatus: 'Открыт' },
  { name: 'Live-queue', status: 'Живая очередь', apiStatus: 'Живая очередь' }
]);

function getAllBookings(table) {
  return [...table.orders, ...table.reservations];
}

const getOrderStartTime = (order) => {
  const tz = props.restaurantData.restaurant.timezone || 'Europe/Moscow';
  const date = parseISO(order.start_time || order.seating_time);
  return isNaN(date) ? '' : formatInTimeZone(date, tz, 'HH:mm');
};

const getOrderEndTime = (order) => {
  const tz = props.restaurantData.restaurant.timezone || 'Europe/Moscow';
  const date = parseISO(order.end_time);
  return isNaN(date) ? '' : formatInTimeZone(date, tz, 'HH:mm');
};

const getOrderFullTime = (order) =>
  `${getOrderStartTime(order)} - ${getOrderEndTime(order)}`;

function getSlotFromTime(time) {
  const [hours, minutes] = time.split(':').map(Number);
  const slotMinutes = minutes < 30 ? '00' : '30';
  return `${String(hours).padStart(2, '0')}:${slotMinutes}`;
}

function getOrdersForTimeSlot(orders, hourSlot) {
  return orders.filter((order) => {
    const time = getOrderStartTime(order);
    const slot = getSlotFromTime(time);
    return slot === hourSlot;
  });
}

function isOverlapping(a, b) {
  const aStart = parseISO(a.start_time || a.seating_time);
  const bStart = parseISO(b.start_time || b.seating_time);

  const diff = Math.abs(bStart - aStart);

  return diff <= 30 * 60 * 1000;
}

function getOverlappingOrdersForSlot(orders) {
  const groups = [];
  for (const order of orders) {
    let added = false;
    for (const group of groups) {
      if (group.some((o) => isOverlapping(o, order))) {
        group.push(order);
        added = true;
        break;
      }
    }
    if (!added) {
      groups.push([order]);
    }
  }
  return groups;
}

function getOrderPositionStyle(order, overlappingGroups, allOrders) {
  const group = overlappingGroups.find((g) => g.includes(order));
  const index = group.indexOf(order);
  const ordersCount = group.length;

  const start = parseISO(order.start_time || order.seating_time);
  const end = parseISO(order.end_time);

  const startMin = start.getHours() * 60 + start.getMinutes();
  const endMin = end.getHours() * 60 + end.getMinutes();

  const offset = (startMin % 30) / 30 * 100;
  const height = ((endMin - startMin) / 30) * 100;
  const width = 100 / ordersCount;

  let marginLeft = '0px';

  if (ordersCount > 1) {
    const groupOverlapsOther = allOrders.some((other) => {
      if (group.includes(other)) return false;
      return group.some((gOrder) => {
        const gStart = new Date(gOrder.start_time || gOrder.seating_time);
        const gEnd = new Date(gOrder.end_time);
        const oStart = new Date(other.start_time || other.seating_time);
        const oEnd = new Date(other.end_time);
        return gStart < oEnd && gEnd > oStart;
      });
    });

    if (groupOverlapsOther) {
      const firstInGroup = group.reduce((earliest, current) => {
        const currentStartTime = current.start_time || current.seating_time;
        const earliestStartTime = earliest.start_time || earliest.seating_time;

        if (!currentStartTime || !earliestStartTime) return earliest;

        return new Date(currentStartTime) < new Date(earliestStartTime)
          ? current
          : earliest;
      }, group[0]);

      if (order.id === firstInGroup.id) {
        marginLeft = '4px';
      }
    }
  } else {
    const overlapLevel = countOverlaps(order, allOrders);
    marginLeft = `${overlapLevel * 4}px`;
  }

  return {
    top: `${offset}%`,
    height: `${height}%`,
    width: `calc(${width}% - ${marginLeft})`,
    left: `${index * width}%`,
    marginLeft
  };
}

function countOverlaps(order, allOrders) {
  const orderStart = new Date(order.start_time || order.seating_time);
  const orderEnd = new Date(order.end_time);

  let overlapCount = 0;

  for (const other of allOrders) {
    if (other.id === order.id) continue;

    const otherStart = new Date(other.start_time || order.seating_time);
    const otherEnd = new Date(other.end_time);

    const overlaps =
      orderStart < otherEnd && orderEnd > otherStart;

    if (overlaps && otherStart < orderStart) {
      overlapCount++;
    }
  }
  return overlapCount;
}
</script>

<template>
  <div class="empty-container" v-if="!(restaurantData.tables && restaurantData.tables.length > 0)">
    <span class="h2">Данных нет</span>
  </div>
  <div class="table-wrapper" v-else>
    <div class="grid-container table-scroll" ref="scrollTarget" 
      :style="`grid-template-columns: 32px repeat(${restaurantData.tables.length}, 80px)`"
    >
      <div class="cell empty"></div>

      <div v-for="table in restaurantData.tables" :key="table" class="cell header-tables">
        <div class="header__first-row">
          <span class="table-number">
            <span class="hash h2">#</span>
            <span class="number">{{ table.number }}</span>
          </span>
          <span class="table-capacity h2">{{ table.capacity }} чел</span>
        </div>
        <div class="header__second-row h2">
          <span class="table-zone">{{ table.zone }}</span>
        </div>
        
      </div>

      <div v-for="(hour, timeIndex) in timeList" :key="timeIndex" class="row-wrapper">

        <div class="cell header-time h2">{{ hour }}</div>

        <div
          v-for="(table, tableIndex) in restaurantData.tables"
          :key="table.id"
          :class="['cell', { 'no-left-border': tableIndex === 0 }]"
        >
          <div
            v-for="order in getOrdersForTimeSlot(table.orders, hour)"
            :key="order.id"
            class="order-chip"
            :class="order.status.toLowerCase()"
            :style="getOrderPositionStyle(order, getOverlappingOrdersForSlot(getAllBookings(table), hour), getAllBookings(table))"
          >
            <div v-for="orderLabel in orderLabels">
              <div class="orderInfo" v-if="order.status === orderLabel.apiStatus">
                <span class="order-name h2-bold">{{ orderLabel.name }}</span>
                <span :class="['order-status h3-bold', orderLabel.apiStatus.toLowerCase()]">{{ orderLabel.status }}</span>
                <span class="order-time h2">{{ getOrderFullTime(order) }}</span>
              </div>
            </div>
          </div>

          <div
            v-for="reservation in getOrdersForTimeSlot(table.reservations, hour)"
            :key="reservation.id"
            class="reservation-chip" 
            :style="getOrderPositionStyle(reservation, getOverlappingOrdersForSlot(getAllBookings(table), hour), getAllBookings(table))"
          >
            <div v-for="reservationLabel in reservationLabels">
              <div 
                class="reservation-info" 
                :class="reservationLabel.name.toLowerCase()"
                v-if="reservation.status === reservationLabel.apiStatus"
              >
                <span class="h3">№111</span>
                <div class="reservation-guests-info">
                  <span class="reservation-name h2">{{ reservation.name_for_reservation }};</span>
                  <div class="reservation-num-people">
                    <span class="reservation-num h2">{{ reservation.num_people }}</span>
                    <span class="h3">чел</span>
                  </div>
                </div>
                <span :class="['reservation-status h3-bold', reservationLabel.name.toLowerCase()]">{{ reservationLabel.status }}</span>
                <div class="reservation-phone-number">
                  <IconPhone style="width: 12px; height: 12px;"/>
                  <span class="phone h2">{{ reservation.phone_number }}</span>
                </div>
                <span class="reservation-time h2">{{ getOrderFullTime(reservation) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.empty-container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.table-wrapper{
  .grid-container {
    display: grid;
    grid-auto-rows: 40px;
    grid-template-rows: 48px;
    min-width: max-content;
    width: auto;

    .cell {
      position: relative;
      border: 1px solid var(--border-color-table);
      display: flex;
      flex-direction: row;
      align-items: flex-start;
      justify-content: center;
    }
    .cell.empty {
      background-color: transparent;
      border: none;
    }
    .cell.header-tables {
      position: sticky;
      top: 0;
      border: none;
      display: flex;
      flex-direction: column;
      height: 48px;
      background: var(--bg-color);
      z-index: 10;

      .header__first-row {
        display: flex;
        flex-direction: row;
        align-items: center;
        margin: 0 auto;
        gap: 4px;
        .table-number {
          text-align: end;

          .number {
            font-weight: 600;
            font-size: 13px;
            line-height: 20px;   
            color: var(--text-color);
          }
        }
      }
      .header__second-row {
        margin: 0 auto;
      }
    }
    .cell.header-time {
      position: sticky;
      left: 0;
      border: none;
      color: var(--text-color-grey-light) !important;
      background: var(--bg-color);
      z-index: 9;
    }
    .cell.no-left-border {
      border-left: none;
    }
    .row-wrapper {
      display: contents;

      .order-chip {
        box-sizing: border-box;
        position: absolute;
        left: 0;
        display: flex;
        flex-direction: row;
        justify-content: flex-start;
        align-items: flex-start;
        padding-left: 0;
        width: 80px;
        min-height: 50%;
        border-radius: 4px;
        overflow: hidden;
        z-index: 1;
        cursor: pointer;

        .orderInfo {
          display: flex;
          flex-direction: column;
          justify-content: flex-start;
          flex-wrap: nowrap;
          padding-left: 6px;

          .order-name {
            margin-top: 2px;
            color: var(--text-color);
          }

          .order-status {
            padding: 2px;
            max-width: max-content;
            border-radius: 4px;
          }
          .order-status.new,
          .order-status.closed {
            background: var(--bg-reservation-status);
          }
          .order-status.bill {
            background: rgba(74, 201, 155, 0.32);
          }

          .order-time {
            color: var(--text-color);
            word-break: keep-all;
          }
        }
      }
      .order-chip:hover {
        backdrop-filter: blur(4px);
        width: 100% !important;
        min-height: 80px !important;
        z-index: 10;
      }
      .reservation-chip {
        box-sizing: border-box;
        position: absolute;
        left: 0;
        display: flex;
        flex-direction: row;
        justify-content: flex-start;
        align-items: flex-start;
        padding-left: 0;
        width: 80px;
        min-height: 50%;
        border-radius: 4px;
        overflow: hidden;
        z-index: 1;
        cursor: pointer;

        .reservation-info {
          display: flex;
          flex-direction: column;
          justify-content: flex-start;
          align-items: flex-start;
          flex-wrap: nowrap;
          padding: 2px 2px 2px 6px;

          .reservation-guests-info {
            display: flex;
            flex-direction: row;
            justify-content: flex-start;
            align-items: flex-start;
            flex-wrap: wrap;
            height: auto;
            
            .reservation-name {
              color: var(--text-color);
              margin-right: 4px;
            }
            .reservation-num-people {
              display: flex;
              justify-content: flex-start;
              align-items: center;
              .reservation-num {
                color: var(--text-color);
              }
            }
          }
          .reservation-status {
            text-wrap: nowrap;
            padding: 2px;
            max-width: max-content;
            border-radius: 4px;
          }
          .reservation-status.new {
            background: rgba(0, 122, 255, 1);
            color: white;
          }
          .reservation-status.waiting {
            color: rgba(0, 151, 253, 1);
            background: rgba(0, 151, 253, 0.1);
          }
          .reservation-status.open {
            color: white;
            background: rgba(74, 201, 155, 0.32);
          }
          .reservation-status.closed {
            color: var(--text-color);
            background: var(--bg-reservation-status);
          }
          .reservation-status.live-queue {
            color: var(--text-color);
            background: var(--bg-reservation-status);
          }
          .reservation-phone-number {
            display: flex;
            justify-content: flex-start;
            align-items: center;
            .phone {
              color: var(--text-color);
            }
          }
          .reservation-time {
            color: var(--text-color);
          }
        }
      }
      .reservation-chip:hover {
        backdrop-filter: blur(4px);
        width: 110px !important;
        min-height: 80px !important;
        z-index: 10;
      }
      .reservation-guests-info.narrow {
        flex-direction: column;
      }     
    }
  }
}
</style>