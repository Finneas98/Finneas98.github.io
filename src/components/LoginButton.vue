<template>
  <div class="flex justify-center mt-6">
    <button v-if="!isAuthenticated" @click="login" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-6 rounded-lg shadow-md transition duration-300">
      Login with Microsoft
    </button>
    <button v-if="isAuthenticated" @click="logout" class="bg-gray-200 hover:bg-gray-300 text-gray-800 font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 text-sm flex items-center gap-2">
      <span>Logout</span>
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, defineEmits } from "vue";
import { PublicClientApplication } from "@azure/msal-browser";

const emit = defineEmits(["auth-changed"]);

const isAuthenticated = ref(false);
const accessToken = ref(null);

const msalConfig = {
  auth: {
    clientId: "c345bebf-7e3e-4b6d-8e4e-7d99bf4397af",
    authority: "https://login.microsoftonline.com/common/v2.0",
    redirectUri: "http://localhost:3000/"
  },
  cache: {
    cacheLocation: "sessionStorage",
    storeAuthStateInCookie: true
  }
};

const msalInstance = new PublicClientApplication(msalConfig);

onMounted(async () => {
  await msalInstance.initialize();
  checkAuthentication();
});

const checkAuthentication = async () => {
  const accounts = msalInstance.getAllAccounts();
  if (accounts.length > 0) {
    isAuthenticated.value = true;
    await getAccessToken();
  }
  emit("auth-changed", { isAuthenticated: isAuthenticated.value, accessToken: accessToken.value });
};

const login = async () => {
  try {
    await msalInstance.loginPopup({ scopes: ["Calendars.Read"] });
    isAuthenticated.value = true;
    await getAccessToken();
    emit("auth-changed", { isAuthenticated: true, accessToken: accessToken.value });
  } catch (error) {
    console.error("Login failed", error);
  }
};

const logout = async () => {
  await msalInstance.logoutPopup();
  isAuthenticated.value = false;
  accessToken.value = null;
  emit("auth-changed", { isAuthenticated: false, accessToken: null });
};

const getAccessToken = async () => {
  try {
    const response = await msalInstance.acquireTokenSilent({
      scopes: ["https://graph.microsoft.com/.default"],
      account: msalInstance.getAllAccounts()[0]
    });
    accessToken.value = response.accessToken;
    emit("auth-changed", { isAuthenticated: true, accessToken: accessToken.value });
  } catch (error) {
    console.error("Error retrieving token", error);
  }
};
</script>
