<template>
   <div
       v-if="isAuthenticated"
       class="bg-orange-300 p-5 rounded-xl shadow-md max-w-[600px] w-full relative"
   >
      <h2 class="text-2xl font-semibold text-gray-700 mb-2">Next Week’s Schedule</h2>
      <h3 class="text-lg font-medium text-gray-600 mb-4">Week Starting: {{ getNextMonday() }}</h3>

      <div class="relative w-full h-full">
         <div
             v-if="Object.keys(groupedEvents).length"
             ref="scrollContainer"
             class="overflow-hidden flex flex-col space-y-4 max-h-[600px]"
         >
            <div
                v-for="(events, date) in groupedEvents"
                :key="date"
                class="flex flex-col space-y-2"
            >
               <!-- Day Divider -->
               <div class="bg-orange-100 text-orange-800 font-semibold px-4 py-2 rounded">
                  {{ date }}
               </div>

               <!-- Events -->
               <div
                   v-for="event in events"
                   :key="event.id"
                   class="bg-gray-100 p-3 rounded-lg shadow w-full"
               >
                  <strong class="text-lg text-gray-800">{{ event.subject }}</strong>
                  <p class="text-gray-600">
                     📅 {{ formatDate(event.start.dateTime) }}
                     🕒 {{ formatTime(event.start.dateTime) }} - {{ formatTime(event.end.dateTime) }}
                  </p>
                  <p class="text-gray-600">
                     📍 {{ event.location.displayName || "No location" }}
                  </p>
                  <p class="text-gray-500">
                     👤 Created by: {{ event.organizer.emailAddress.name }}
                  </p>
               </div>
            </div>
         </div>

         <div v-else>
            <p>No events this week.</p>
         </div>

         <!-- Fade-out overlay -->
         <div
             v-if="filteredEvents.length > 4"
             class="pointer-events-none absolute bottom-0 left-0 right-0 h-12 bg-gradient-to-t from-blue-100 to-transparent"
         ></div>
      </div>
   </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';

export default {
   props: ['isAuthenticated', 'events'],
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
            const isNotCancelled = event.isCancelled !== 'true';
            return isWithinRange && isNotCancelled;
         });
      });

      const groupedEvents = computed(() => {
         const groups = {};

         filteredEvents.value.forEach(event => {
            const dateKey = new Date(event.start.dateTime).toLocaleDateString(undefined, {
               weekday: 'long',
               year: 'numeric',
               month: 'short',
               day: 'numeric'
            });

            if (!groups[dateKey]) {
               groups[dateKey] = [];
            }

            groups[dateKey].push(event);
         });

         // Sort each day's events by time
         Object.keys(groups).forEach(dateKey => {
            groups[dateKey].sort((a, b) => new Date(a.start.dateTime) - new Date(b.start.dateTime));
         });

         // Sort the keys chronologically
         const sortedKeys = Object.keys(groups).sort((a, b) => {
            return new Date(groups[a][0].start.dateTime) - new Date(groups[b][0].start.dateTime);
         });

         const sortedGroups = {};
         sortedKeys.forEach(key => {
            sortedGroups[key] = groups[key];
         });

         return sortedGroups;
      });

      watch(filteredEvents, (events) => {
         console.log('next week events length: ' + events.length);
         if (events.length > 4 && scrollContainer.value) {
            if (scrollInterval) clearInterval(scrollInterval);

            scrollInterval = setInterval(() => {
               if (scrollContainer.value) {
                  scrollContainer.value.scrollTop += 1;

                  if (
                      scrollContainer.value.scrollTop >=
                      scrollContainer.value.scrollHeight - scrollContainer.value.clientHeight
                  ) {
                     scrollContainer.value.scrollTop = 0;
                  }
               }
            }, 50);
         }
      }, { immediate: true });

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

      const formatDate = (date) =>
          new Date(date).toLocaleDateString('en-GB', { timeZone: 'Europe/Dublin' });

      const formatTime = (dateString) => {
         const date = new Date(dateString + 'Z'); // Add Z to make it UTC
         return date.toLocaleTimeString('en-GB', {
            hour: '2-digit',
            minute: '2-digit',
            timeZone: 'Europe/Dublin',
         });
      };

      return {
         scrollContainer,
         filteredEvents,
         groupedEvents,
         getNextMonday,
         formatDate,
         formatTime,
      };
   }
};
</script>

<style scoped>
</style>
