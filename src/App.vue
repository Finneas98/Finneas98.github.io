<template>
   <div class="flex flex-row min-h-screen w-full gap-10 justify-center items-center py-10">

      <!-- Left Column: Logo + Login -->
      <div class="flex flex-col items-center space-y-10 w-1/4">
         <img
             alt="FMCI logo"
             class="max-w-full h-auto"
             src="./assets/fmci-logo-transparent.png"
         />

         <div v-if="showLogin" class="flex flex-col items-center space-y-4 w-full">
            <LoginButton @auth-changed="handleAuthChange" />
            <button
                class="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 transition w-full"
                @click="hideLoginSection"
            >
               Hide Login Button
            </button>
         </div>
      </div>

      <!-- Center Column: Current Week -->
      <div class="flex flex-col items-center w-1/4">
         <CurrentWeekSchedule
             v-if="isAuthenticated"
             :isAuthenticated="isAuthenticated"
             :accessToken="accessToken"
             :events="currentWeekEvents"
         />
      </div>

      <!-- Right Column: Next Week -->
      <div class="flex flex-col items-center w-1/4">
         <NextWeekSchedule
             v-if="isAuthenticated"
             :isAuthenticated="isAuthenticated"
             :accessToken="accessToken"
             :events="nextWeekEvents"
         />
      </div>

   </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import CurrentWeekSchedule from "@/components/CurrentWeekSchedule.vue";
import NextWeekSchedule from "@/components/NextWeekSchedule.vue";
import LoginButton from "@/components/LoginButton.vue";
import axios from "axios";

const currentWeekEvents = ref([]);
const nextWeekEvents = ref([]);

const isAuthenticated = ref(false);
const accessToken = ref(null);
const showLogin = ref(true);

const fetchEvents = async () => {
   if (!accessToken.value) return;

   try {
      const today = new Date(); // only once!

      // === Current Week ===
      const dayOfWeek = today.getDay();
      const startOfWeek = new Date(today);
      startOfWeek.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1));
      startOfWeek.setHours(0, 0, 0, 0);

      const endOfWeek = new Date(startOfWeek);
      endOfWeek.setDate(startOfWeek.getDate() + 4);
      endOfWeek.setHours(23, 59, 59, 999);

      const responseCurrentWeek = await axios.get(
          `https://graph.microsoft.com/v1.0/me/calendarView?startDateTime=${startOfWeek.toISOString()}&endDateTime=${endOfWeek.toISOString()}`,
          { headers: { Authorization: `Bearer ${accessToken.value}` } }
      );
      currentWeekEvents.value = responseCurrentWeek.data.value;

      // === Next Week ===
      const nextMonday = new Date(today);
      nextMonday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1) + 7);
      nextMonday.setHours(0, 0, 0, 0);

      const nextFriday = new Date(nextMonday);
      nextFriday.setDate(nextMonday.getDate() + 4);
      nextFriday.setHours(23, 59, 59, 999);

      const responseNextWeek = await axios.get(
          `https://graph.microsoft.com/v1.0/me/calendarView?startDateTime=${nextMonday.toISOString()}&endDateTime=${nextFriday.toISOString()}`,
          { headers: { Authorization: `Bearer ${accessToken.value}` } }
      );
      nextWeekEvents.value = responseNextWeek.data.value;

   } catch (error) {
      console.error("Error fetching events", error);
   }
};

let intervalId;

onMounted(() => {
   intervalId = setInterval(fetchEvents, 60000); // poll every 60 seconds
});

onUnmounted(() => {
   clearInterval(intervalId);
});

const handleAuthChange = ({ isAuthenticated: authState, accessToken: token }) => {
   isAuthenticated.value = authState;
   accessToken.value = token;

   if (authState && token) {
      fetchEvents(); // 💥 fetch immediately after login
   }
};

const hideLoginSection = () => {
   showLogin.value = false;
};
</script>

<style scoped>
/* No styles needed now — all handled by Tailwind */
</style>
