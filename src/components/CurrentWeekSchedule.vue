<template>
   <div
       v-if="isAuthenticated"
       class="bg-blue-300 p-5 rounded-xl shadow-md max-w-[500px] w-full relative h-full"
   >
      <h2 class="text-2xl font-semibold text-gray-700 mb-2">This Week’s Schedule</h2>
      <h3 class="text-lg font-medium text-gray-600 mb-4">Week Starting: {{ getMonday() }}</h3>
      <div class="relative w-full">
         <!-- Scrollable event container -->
         <div
             v-if="Object.keys(groupedEvents).length > 0"
             ref="scrollContainer"
             class="overflow-hidden flex flex-col space-y-4 max-h-[600px]"
         >
            <div
                v-for="(events, date) in groupedEvents"
                :key="date"
                class="flex flex-col space-y-2"
            >
               <!-- Day Divider -->
               <div class="bg-blue-100 text-blue-800 font-semibold px-4 py-2 rounded">
                  {{ date }}
               </div>
               <!-- Events for the Day -->
               <div
                   v-for="event in events"
                   :key="event.id"
                   :class="[
                       'p-3 rounded-lg shadow w-full',
                       isPastEvent(event) ? 'bg-gray-300 text-gray-500' : 'bg-gray-100',
                       isOngoingEvent(event) ? 'bg-green-300' : 'bg-gray-100'
                       ]"
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
         </div>
         <div v-else class="flex flex-col items-center justify-center h-40 bg-blue-100 rounded-lg shadow-inner text-blue-800">
            <svg class="w-12 h-12 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
               <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                     d="M9.75 17L3 10.25l1.5-1.5L9.75 14l10.5-10.5L21.75 5.25z" />
            </svg>
            <p class="text-lg font-medium">No events planned this week</p>
         </div>
         <!-- Fade-out overlay -->
         <div
             v-if="filteredEvents.length > 4"
             class="pointer-events-none absolute bottom-0 left-0 right-0 h-12 bg-gradient-to-t from-blue-100 to-transparent">
         </div>
      </div>
   </div>
</template>


<script>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';

export default {
   props: ['isAuthenticated', 'events'],
   methods: {
      isPastEvent(event) {
         const eventEnd = new Date(event.end.dateTime + 'Z');
         const now = new Date();
         return now > eventEnd;
      },
      isOngoingEvent(event) {
         const eventEnd = new Date(event.end.dateTime + 'Z');
         const eventStart = new Date(event.start.dateTime + 'Z');
         const now = new Date();
         return (now > eventStart) && (now < eventEnd);
      }
   },
   setup(props) {
      const scrollContainer = ref(null);
      let scrollInterval;

      const filteredEvents = computed(() => {
         const today = new Date();
         const dayOfWeek = today.getDay();
         const monday = new Date(today);
         monday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1));
         monday.setHours(0, 0, 0, 0);

         const friday = new Date(monday);
         friday.setDate(monday.getDate() + 4);
         friday.setHours(23, 59, 59, 999);

         return props.events.filter(event => {
            const eventDate = new Date(event.start.dateTime);
            const isWithinRange = eventDate >= monday && eventDate <= friday;
            const isNotCancelled = event.isCancelled !== true;
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

         // Sort events within each group by start time
         Object.keys(groups).forEach(dateKey => {
            groups[dateKey].sort((a, b) => new Date(a.start.dateTime) - new Date(b.start.dateTime));
         });

         // Sort the day keys by the earliest event's date
         const sortedKeys = Object.keys(groups).sort((a, b) => {
            return new Date(groups[a][0].start.dateTime) - new Date(groups[b][0].start.dateTime);
         });

         // Return a new object with sorted keys and sorted events
         const sortedGroups = {};
         sortedKeys.forEach(key => {
            sortedGroups[key] = groups[key];
         });

         return sortedGroups;
      });


      // Auto scroll logic
      watch(filteredEvents, (events) => {
         console.log('this week events length: ' + events.length);
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

      const getMonday = () => {
         const today = new Date();
         const dayOfWeek = today.getDay();
         const monday = new Date(today);
         monday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1));
         return monday.toLocaleDateString('en-GB');
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
         getMonday,
         formatDate,
         formatTime,
      };
   }
};
</script>

<style scoped>
</style>
