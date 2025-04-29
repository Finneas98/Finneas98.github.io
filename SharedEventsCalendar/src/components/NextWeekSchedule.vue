<template>
   <div v-if="isAuthenticated" class="bg-blue-100 p-5 rounded-xl shadow-md max-w-[600px] w-full relative">
      <h2 class="text-2xl font-semibold text-gray-700 mb-2">Next Week’s Schedule</h2>
      <h3 class="text-lg font-medium text-gray-600 mb-4">Week Starting: {{ getNextMonday() }}</h3>

      <!-- Make this relative so fade can be absolutely positioned inside -->
      <div class="relative w-full h-full">

         <!-- Scrollable area -->
         <div
             v-if="filteredEvents"
             ref="scrollContainer"
             class="overflow-hidden flex flex-col space-y-4 max-h-[600px]">
            <div
                v-for="event in filteredEvents"
                :key="event.id"
                class="bg-gray-100 p-3 rounded-lg shadow w-full"
            >
               <strong class="text-lg text-gray-800">{{ event.subject }}</strong>
               <p class="text-gray-600">
                  📅 {{ formatDate(event.start.dateTime) }}
                  🕒 {{ formatTime(event.start.dateTime) }} - {{ formatTime(event.end.dateTime) }}
               </p>
               <p class="text-gray-600">📍 {{ event.location.displayName || "No location" }}</p>
               <p class="text-gray-500">👤 Created by: {{ event.organizer.emailAddress.name }}</p>
            </div>
         </div>

         <div
             v-else
             class=""
         >
            <p> No events this week. </p>
         </div>

         <!-- Fade-out overlay: OUTSIDE scroll container, inside relative wrapper -->
         <div
             v-if="filteredEvents.length > 4"
             class="pointer-events-none absolute bottom-0 left-0 right-0 h-12 bg-gradient-to-t from-blue-100 to-transparent">
         </div>

      </div>
   </div>
</template>


<script>
import {ref, onMounted, onUnmounted, computed} from 'vue';

export default {
   props: ["isAuthenticated", "events"],
   setup(props) {
      const scrollContainer = ref(null);
      let scrollInterval;

      const filteredEvents = computed(() => {
         const today = new Date();
         const dayOfWeek = today.getDay();

         const nextMonday = new Date(today);
         nextMonday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1) + 7);
         nextMonday.setHours(0, 0, 0, 0);

         const nextFriday = new Date(nextMonday);
         nextFriday.setDate(nextMonday.getDate() + 4);
         nextFriday.setHours(23, 59, 59, 999);

         return props.events.filter(event => {
            const eventDate = new Date(event.start.dateTime);
            const isWithinRange = eventDate >= nextMonday && eventDate <= nextFriday;
            const isNotCancelled = event.isCancelled !== "true"; // <-- Important
            return isWithinRange && isNotCancelled;
         });
      });

      onMounted(() => {
         console.log("next week events length: " + filteredEvents.value.length);
         if (filteredEvents.value.length > 4) {
            scrollInterval = setInterval(() => {
               if (scrollContainer.value) {
                  scrollContainer.value.scrollTop += 1;

                  // Check if reached the bottom
                  if (scrollContainer.value.scrollTop >= scrollContainer.value.scrollHeight - scrollContainer.value.clientHeight) {
                     scrollContainer.value.scrollTop = 0; // Reset to top!
                  }
               }
            }, 50);
         }
      });

      onUnmounted(() => {
         clearInterval(scrollInterval);
      });

      const getNextMonday = () => {
         const today = new Date();
         const dayOfWeek = today.getDay();
         const nextMonday = new Date(today);
         nextMonday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1) + 7);
         return nextMonday.toLocaleDateString('en-GB');
      };

      const formatDate = (date) => new Date(date).toLocaleDateString('en-GB');
      const formatTime = (date) => new Date(date).toLocaleTimeString('en-GB', {hour: '2-digit', minute: '2-digit'});

      return {
         scrollContainer,
         filteredEvents,
         getNextMonday,
         formatDate,
         formatTime
      };
   }
};
</script>

<style scoped>
</style>
