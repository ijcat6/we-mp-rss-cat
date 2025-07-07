<template>
  <a-layout class="app-container">
    <a-layout-header class="app-header" v-show="hasLogined">
      <div class="header-left">
        <div class="logo">
          <img :src="logo" alt="avatar" :width="60" style="margin-right:1rem;">
          <router-link to="/">{{ APP_TITLE }}</router-link>
          <a-tooltip content="点我扫码授权" position="bottom">
            <icon-scan @click="showAuthQrcode()" style="margin-left: 10px; cursor: pointer;color: #000;" />
          </a-tooltip>
        </div>
      </div>
      <div class="header-right">
        <a-link :href="DOCS_URL" target="_blank" style="margin-right: 20px;">Docs</a-link>
        <a-link :href="GITEE_URL" target="_blank" style="margin-right: 20px;">Gitee</a-link>
        <a-link :href="GITHUB_URL" target="_blank" style="margin-right: 20px;">GitHub</a-link>

        <a-tooltip content="如果您需要部署此项目，建议采用腾讯云服务器，您懂得" position="bottom">
          <a-link href="https://cloud.tencent.com/act/cps/redirect?redirect=2446&cps_key=f8ce741e7b24cd68141ab2115122ea94&from=console" target="_blank" style="margin-right: 20px;">云部署</a-link>
        </a-tooltip>
        <a-tooltip content="您的支持是作者的最大动力，来一杯咖啡吧" position="bottom">
          <a-link href="https://www.paypal.com/ncp/payment/PUA72WYLAV5KW" target="_blank" style="margin-right: 20px;">赞助</a-link>
        </a-tooltip>

        <a-link @click="showSponsorModal" style="margin-right: 20px; cursor: pointer;" type="text">支持</a-link>

        <a-dropdown position="br" trigger="click">
          <div class="user-info">
            <a-avatar :size="36">
              <img v-if="userInfo.avatar" :src="userInfo.avatar" alt="avatar">
              <icon-user v-else />
            </a-avatar>
            <span class="username">{{ userInfo.username }}</span>
          </div>
          <template #content>
            <a-doption @click="goToEditUser">
              <template #icon><icon-user /></template>
              个人中心
            </a-doption>
            <a-doption @click="goToChangePassword">
              <template #icon><icon-lock /></template>
              修改密码
            </a-doption>
            <a-doption @click="showAuthQrcode">
              <template #icon><icon-scan /></template>
              扫码授权
            </a-doption>
            <a-doption @click="handleLogout">
              <template #icon><icon-user /></template>
              退出登录
            </a-doption>
          </template>
        </a-dropdown>
        <WechatAuthQrcode ref="qrcodeRef" />
        <a-modal v-model:visible="sponsorVisible" title="感谢支持" :footer="false" unmount-on-close>
          <div style="text-align: center;">
            <p>如果您觉得这个项目对您有帮助,请给Rachel来一杯Coffee吧~ </p>
            <img src="@/assets/images/sponsor.jpg" alt="赞赏码" style="max-width: 300px; margin-top: 20px;">
          </div>
        </a-modal>
      </div>
    </a-layout-header>

    <a-layout>
      <a-layout>
        <a-layout-content class="app-content">
          <router-view />
        </a-layout-content>
      </a-layout>
    </a-layout>
  </a-layout>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, provide } from 'vue';
import { useRouter, useRoute } from 'vue-router';
import { Message } from '@arco-design/web-vue';
import { getCurrentUser, logout } from '@/api/auth';
import WechatAuthQrcode from '@/components/WechatAuthQrcode.vue';

// --- Performance Optimizations ---

// 1. Move static values outside the setup function to prevent them from being re-declared on each component instance.
const APP_TITLE = import.meta.env.VITE_APP_TITLE || '微信公众号订阅助手';
const DOCS_URL = '/api/docs';
const GITEE_URL = 'https://gitee.com/rachel_os/we-mp-rss';
const GITHUB_URL = 'https://github.com/rachelos/we-mp-rss';

const router = useRouter();
const route = useRoute();

const logo = ref("/static/logo.svg");
const userInfo = ref({ username: '', avatar: '' });
const hasLogined = ref(!!localStorage.getItem('token'));
const sponsorVisible = ref(false);
const qrcodeRef = ref();

// --- Methods ---

const fetchUserInfo = async () => {
  // Only fetch if user info is not already loaded
  if (userInfo.value.username) return;
  try {
    userInfo.value = await getCurrentUser();
  } catch (error) {
    console.error('获取用户信息失败', error);
    // If fetching user info fails, it might indicate an invalid token.
    await handleLogout();
  }
};

const showAuthQrcode = () => {
  qrcodeRef.value?.startAuth();
};
provide('showAuthQrcode', showAuthQrcode);

const showSponsorModal = (e: Event) => {
  e.preventDefault();
  sponsorVisible.value = true;
};

const goToEditUser = () => router.push({ name: 'EditUser' });
const goToChangePassword = () => router.push({ name: 'ChangePassword' });

const handleLogout = async () => {
  try {
    await logout();
  } catch (error) {
    console.error('Logout API call failed', error);
  } finally {
    localStorage.removeItem('token');
    hasLogined.value = false;
    userInfo.value = { username: '', avatar: '' }; // Clear user info
    router.push('/login');
    Message.success('已退出登录');
  }
};

// --- Lifecycle and Watchers ---

onMounted(() => {
  if (hasLogined.value) {
    fetchUserInfo();
  }
});

// 2. Watch for route changes to handle login status, but with more precise control.
watch(
  () => route.path,
  (newPath) => {
    const stillLoggedIn = !!localStorage.getItem('token');
    if (hasLogined.value !== stillLoggedIn) {
      hasLogined.value = stillLoggedIn;
    }

    // If the user becomes logged in (e.g., after the login page), fetch their info.
    if (hasLogined.value && !userInfo.value.username) {
      fetchUserInfo();
    }
    
    // If user navigates to login page, ensure logged-in state is false
    if (newPath === '/login') {
        hasLogined.value = false;
    }
  },
  { immediate: true } // Run watcher immediately on component mount
);
</script>

<style scoped>
.app-container {
  min-height: 100vh;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px;
  height: 64px;
  background: var(--color-bg-2);
  border-bottom: 1px solid var(--color-border);
}

.header-left {
  display: flex;
  align-items: center;
}

.logo {
  display: flex;
  align-items: center;
  font-size: 18px;
  font-weight: 500;
}

.logo svg {
  margin-right: 10px;
  font-size: 24px;
  color: var(--primary-color);
}

.header-right {
  display: flex;
  align-items: center;
}

.user-info {
  display: flex;
  align-items: center;
  cursor: pointer;
}

.username {
  margin-left: 10px;
}

.app-content {
  background: var(--color-bg-1);
  min-height: calc(100vh - 64px);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
