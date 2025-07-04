<template>
  <h1 class="text-5xl font-roboto py-24 text-white text-center mt-8">Overview of Connected Raspberry Pis</h1>
  <div class="flex flex-col gap-8 px-8 pb-24">
    <div class="bg-white text-gray-800 rounded-3xl p-8 w-full h-full overflow-y-scroll">
      <h2 class="text-2xl font-semibold mb-4">Raspberry Pi Status</h2>
      <table class="min-w-full divide-y divide-gray-700 bg-white">
        <thead class="bg-indigo-50 text-gray-700 sticky top-0">
          <tr>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Select</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">MAC Address</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Status</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Last Connection</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Location</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Failures (Last 24h)</th>
            <th class="px-6 py-3 text-left text-sm font-medium uppercase">Last hour</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-gray-300 text-gray-800">
          <tr v-for="rpi in raspberries" :key="rpi.id" class="hover:bg-gray-100 cursor-pointer"
            @click="navigateToRaspberry(rpi.id)">
            <td class="px-6 py-4">
              <input type="checkbox" :value="rpi.id" v-model="selectedRaspberries"
                class="form-checkbox h-7 w-7 text-indigo-600" @click.stop />
            </td>
            <td class="px-6 py-4">{{ rpi.mac }}</td>
            <td v-if="is_online(rpi)" class="px-6 py-4">
              <span class="inline-block px-2 py-1 text-xs font-semibold text-green-800 bg-green-100 rounded-full">
                Online
              </span>
            </td>
            <td v-else class="px-6 py-4">
              <span class="inline-block px-2 py-1 text-xs font-semibold text-red-800 bg-red-100 rounded-full">
                Offline
              </span>
            </td>
            <td class="px-6 py-4"> {{ rpi.ultimo_registro }}</td>

            <td v-if="rpi.location !== null">
              <span class="px-6 py-4"> {{ rpi.location }} </span>
            </td>
            <td v-else>
              <span class="px-6 py-4">No Location Given </span>
            </td>

            <td class="px-6 py-4">
              <span v-if="!is_online(rpi) || rpi.has_errorDay" class="inline-block w-4 h-4 bg-red-500 rounded-full"
                title="Error"></span>
              <span v-else class="inline-block w-4 h-4 bg-green-500 rounded-full" title="No Error"></span>
            </td>

            <td class="px-6 py-4">
              <span v-if="!is_online(rpi) || rpi.has_errorHour" class="inline-block w-4 h-4 bg-red-500 rounded-full"
                title="Error"></span>
              <span v-else class="inline-block w-4 h-4 bg-green-500 rounded-full" title="No Error"></span>
            </td>
          </tr>
        </tbody>
      </table>

      <div class="p-4 mt-4 bg-blue-50 border border-blue-300 text-blue-800 rounded-md">
        <p> ℹ️ Select 1 raspberry to view failures.</p>
      </div>
      <div class="p-4 mt-1 bg-blue-50 border border-blue-300 text-blue-800 rounded-md">
        <p> ℹ️ Select at least 1 raspberries and no more than 3.</p>
      </div>


      <div class="mt-4 flex justify-end">

        <button @click="openPopup(selectedRaspberries[0])" :disabled="selectedRaspberries.length !== 1" class="px-6 py-2 rounded-lg shadow transition duration-300 ease-in-out
            hover:scale-105
            disabled:bg-transparent disabled:text-gray-400 disabled:border disabled:border-gray-400
            bg-indigo-600 text-white hover:bg-indigo-700 mr-2">
          View Failures
        </button>

        <button @click="viewSelectedData" :disabled="selectedRaspberries.length === 0 || selectedRaspberries.length > 3"
          class="px-6 py-2 rounded-lg shadow transition duration-300 ease-in-out
            hover:scale-105
            disabled:bg-transparent disabled:text-gray-400 disabled:border disabled:border-gray-400
            bg-indigo-600 text-white hover:bg-indigo-700">
          View Selected Data
        </button>

      </div>
    </div>


    <div v-if="isPopupVisible" class="fixed inset-0 bg-gray-800 bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white rounded-lg p-6 w-1/3">
        <h2 class="text-xl font-semibold mb-4">Raspberry Pi Failures</h2>
        <div v-if="getRaspberryById(selectedRaspberries[0]) && !is_online(getRaspberryById(selectedRaspberries[0]))"
          class="px-6 py-4">
          Raspberry Pi {{ selectedRaspberries[0] }} is offline.
        </div>

        <div v-else class="px-6 py-4">
          <h3 class="text-lg font-medium mb-2">Failures last hour:</h3>
          <ul v-if="Object.keys(selectedFailures).length > 0">
            <li v-for="failure in selectedFailures" :key="failureType">
              <strong>
                <template v-if="failure === 1">High Latency</template>
                <template v-else-if="failure === 2">Low Download Speed</template>
                <template v-else-if="failure === 3">Low Upload Speed</template>
                <template v-else-if="failure === 4">Packet Loss</template>
                <template v-else-if="failure === 5">High RTT</template>
              </strong>
            </li>
          </ul>
          <p v-else>No failures in the last hour.</p>
          <br>
          <h3 class="text-lg font-medium mb-2">Failures last day:</h3>
          <ul v-if="Object.keys(selectedFailuresDay).length > 0">
            <li v-for="failure in Object.keys(selectedFailuresDay)" :key="failureType">
              <strong>
                <template v-if="failure === 1">High Latency</template>
                <template v-else-if="failure === 2">Low Download Speed</template>
                <template v-else-if="failure === 3">Low Upload Speed</template>
                <template v-else-if="failure === 4">Packet Loss</template>
                <template v-else-if="failure === 5">High RTT</template>
              </strong>
            </li>
          </ul>
          <p v-else>No failures in the last day.</p>
        </div>


        <button @click="closePopup" class="mt-4 px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700">
          Close
        </button>
      </div>
    </div>


    <div class="bg-white text-gray-800 rounded-3xl shadow-xl p-8 w-full h-full">
      <h2 class="text-2xl font-semibold mb-4">Failures</h2>

      <!-- Scrollable wrapper -->
      <div class="max-h-[400px] overflow-y-auto">
        <table class="min-w-full divide-y divide-gray-700 bg-white">
          <thead class="bg-indigo-50 text-gray-700 sticky top-0">
            <tr>
              <th @click="toggleSort('raspberrypi_id')"
                class="px-6 py-3 text-left text-sm font-medium uppercase cursor-pointer select-none">
                Raspberry Id
              </th>
              <th @click="toggleSort('failure')"
                class="px-6 py-3 text-left text-sm font-medium uppercase cursor-pointer select-none">
                Failure
              </th>
              <th @click="toggleSort('timestamp')"
                class="px-6 py-3 text-left text-sm font-medium uppercase cursor-pointer select-none">
                Time
              </th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-300 text-gray-800">
            <tr v-for="f in sortedFailures" :key="f">
              <td class="px-6 py-4">{{ f.raspberrypi_id }}</td>
              <td class="px-6 py-4">
                <span v-if="f.failure === 1">High Latency</span>
                <span v-else-if="f.failure === 2">Low Download Speed</span>
                <span v-else-if="f.failure === 3">Low Upload Speed</span>
                <span v-else-if="f.failure === 4">Packet Loss</span>
                <span v-else-if="f.failure === 5">High RTT</span>
              </td>
              <td class="px-6 py-4">{{ f.timestamp }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div v-if="graphsVisible" class="mt-6 space-y-6">
      <iframe
        :src="`http://192.92.147.85:3000/d-solo/eehucflethslca/graficos-iniciais?orgId=1&from=now-24h&to=now&${raspberriesQuery}&panelId=5&__feature.dashboardSceneSolo`" 
        class="w-full h-96 rounded-lg shadow-lg"></iframe>

      <iframe
        :src="`http://192.92.147.85:3000/d-solo/eehucflethslca/graficos-iniciais?orgId=1&from=now-24h&to=now&${raspberriesQuery}&panelId=1&__feature.dashboardSceneSolo`"
        class="w-full h-96 rounded-lg shadow-lg"></iframe>

      <iframe
        :src="`http://192.92.147.85:3000/d-solo/eehucflethslca/graficos-iniciais?orgId=1&from=now-24h&to=now&${raspberriesQuery}&panelId=2&__feature.dashboardSceneSolo`" 
        class="w-full h-96 rounded-lg shadow-lg"></iframe>


      <iframe
        :src="`http://192.92.147.85:3000/d-solo/eehucflethslca/graficos-iniciais?orgId=1&from=now-24h&to=now&${raspberriesQuery}&panelId=4&__feature.dashboardSceneSolo`" 
        class="w-full h-96 rounded-lg shadow-lg"></iframe>

      <iframe
        :src="`http://192.92.147.85:3000/d-solo/eehucflethslca/graficos-iniciais?orgId=1&from=now-24h&to=now&${raspberriesQuery}&panelId=6&__feature.dashboardSceneSolo`"
        width="100%" height="330" frameborder="0" class="rounded-lg shadow-lg"></iframe>

    </div>
  </div>
</template>

<script setup>
import { useRouter } from 'vue-router';
import { ref, onMounted, computed } from 'vue';
import { fetchNullRaspberries, fetchRaspberryPis, fetchFailuresLastDay, fetchFailuresLastHour, fetchAllFailures } from '@/services/raspberryService';
import { toast } from 'vue3-toastify';

const nullIds = ref([]);
const raspberries = ref([]);
const router = useRouter();
const selectedRaspberries = ref([]);
const graphsVisible = ref(false);
const previousStatuses = ref({});
const showViewDataNotification = ref(false);
const failures = ref([])
const sortKey = ref('raspberrypi_id');
const sortAsc = ref(true);

const raspberriesQuery = computed(() => {
  if (!selectedRaspberries.value.length) return '';
  return selectedRaspberries.value
    .slice(0, 3)
    .map(id => `var-raspberry=${encodeURIComponent(id)}`)
    .join('&');
});

const toggleSort = (key) => {
  if (sortKey.value === key) {
    sortAsc.value = !sortAsc.value;
  } else {
    sortKey.value = key;
    sortAsc.value = true;
  }
};

const sortedFailures = computed(() => {
  return [...failures.value].sort((a, b) => {
    const aVal = a[sortKey.value];
    const bVal = b[sortKey.value];

    if (sortKey.value === 'timestamp') {
      return sortAsc.value
        ? new Date(aVal) - new Date(bVal)
        : new Date(bVal) - new Date(aVal);
    }

    if (aVal < bVal) return sortAsc.value ? -1 : 1;
    if (aVal > bVal) return sortAsc.value ? 1 : -1;
    return 0;
  });
});



function getRaspberryById(id) {
  return raspberries.value.find(r => r.id === id);
}

function is_online(rpi) {
  const time = new Date(rpi.ultimo_registro).getTime();
  const currTime = new Date().getTime();
  return currTime - time < 8 * 60 * 1000;
}

function navigateToRaspberry(id) {
  router.push({ name: 'raspberry-details', params: { id } });
}

function viewSelectedData() {
  graphsVisible.value = selectedRaspberries.value.length > 0;
  notifyUser();
}


async function refreshRaspberryData() {
  const response = await fetchRaspberryPis();
  raspberries.value = response;


  for (const rpi of raspberries.value) {
    rpi.has_errorHour = await checkLastHourRecords(rpi.id);
    rpi.has_errorDay = await checkLastDayRecords(rpi.id);
    rpi.failuresHour = await getSpecificFailures(rpi.id, "hour");
    rpi.failuresDay = await getSpecificFailures(rpi.id, "day");
  }
}


// ----- check failures -----

async function checkLastHourRecords(raspberryId) {
  try {
    const failures = await fetchFailuresLastHour();
    const hasFailures = Object.values(failures)
      .filter(failureList => Array.isArray(failureList))
      .some((failureList) => failureList.includes(raspberryId));
    return hasFailures; // true -> exist failures, false -> no failures
  } catch (error) {
    return false;
  }
}

async function checkLastDayRecords(raspberryId) {
  try {
    const failures = await fetchFailuresLastDay();
    const hasFailures = Object.values(failures || {}).some((failureList) => {
      if (!Array.isArray(failureList) || failureList.length === 0) {
        return false;
      }
      return failureList.includes(raspberryId);
    });
    return hasFailures; // true -> exist failures, false -> no failures
  } catch (error) {
    console.error(`Error checking last day records for Raspberry Pi ${raspberryId}:`, error);
    return false;
  }
}

async function getSpecificFailures(raspberryId, period) {
  try {
    let failures;
    if (period === "hour") {
      failures = await fetchFailuresLastHour();
    } else if (period === "day") {
      failures = await fetchFailuresLastDay();
    }

    const specificFailures = {};
    for (const [failureType, failureList] of Object.entries(failures)) {
      if (Array.isArray(failureList) && failureList.map(Number).includes(Number(raspberryId))) {
        specificFailures[failureType] = failureList;
      }
    }

    return specificFailures;
  } catch (error) {
    console.error(`Error fetching specific failures for Raspberry Pi ${raspberryId}:`, error);
    return {};
  }
}


const isPopupVisible = ref(false);
const selectedFailures = ref({});
const selectedFailuresDay = ref({});

async function openPopup(raspberryId) {
  selectedFailures.value = await getSpecificFailures(raspberryId, "hour");
  selectedFailuresDay.value = await getSpecificFailures(raspberryId, "day");
  console.log("selectedFailures:", selectedFailures.value);
  isPopupVisible.value = true;
}

function closePopup() {
  isPopupVisible.value = false;
}



// ----- notifications -----

const notifyUser = () => {
  showViewDataNotification.value = true;
  setTimeout(() => {
    showViewDataNotification.value = false;
  }, 3000);
  toast.success('Graphs are loading, see them below.');
};

function checkStatusChange() {
  raspberries.value.forEach((rpi) => {
    const currentStatus = is_online(rpi);
    const previousStatus = previousStatuses.value[rpi.id];

    if (previousStatus === true && currentStatus === false) {
      toast.error(`Raspberry Pi ${rpi.mac} is now offline!`);
    }

    previousStatuses.value[rpi.id] = currentStatus;
  });
}

onMounted(async () => {
  nullIds.value = await fetchNullRaspberries();
  let response = await fetchRaspberryPis();
  raspberries.value = response;

  response = await fetchAllFailures();
  failures.value = response;


  for (const rpi of raspberries.value) {
    rpi.has_errorHour = await checkLastHourRecords(rpi.id);
    rpi.has_errorDay = await checkLastDayRecords(rpi.id);
    rpi.failuresHour = await getSpecificFailures(rpi.id, "hour");
    rpi.failuresDay = await getSpecificFailures(rpi.id, "day");
  }

  raspberries.value.forEach((rpi) => {
    previousStatuses.value[rpi.id] = is_online(rpi);
  });

  setInterval(async () => {
    await refreshRaspberryData();
    checkStatusChange();
  }, 5000);
});
</script>
