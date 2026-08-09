<template>
  <t-config-provider :global-config="getComponentsLocale">
    <router-view />
    <disclaimer-view type="init" />
  </t-config-provider>
</template>

<script setup lang="ts">
import type { ISetup } from '@shared/config/tblSetting';
import { setupObj as tblSetup } from '@shared/config/tblSetting';
import { THEME } from '@shared/config/theme';
import { onMounted, ref, watch, onUnmounted } from 'vue';

import { fetchSetup } from '@/api/setting';
import { useLocale } from '@/locales/useLocale';
import DisclaimerView from '@/pages/Disclaimer.vue';
import { usePlayerStore, useSettingStore } from '@/store';
import { start as startOSpy, stop as stopOSpy } from '@/utils/ospy';
import { start as startVitals, stop as stopVitals } from '@/utils/vitalsObserver';

const storePlayer = usePlayerStore();
const storeSetting = useSettingStore();

const { getComponentsLocale } = useLocale();

const setupConf = ref<ISetup>(tblSetup);

const active = ref({
  disclaimer: false,
});

// ============================================================
//  主题插件通信模块
// ============================================================

const PLUGIN_STYLE_ID = 'zyfun-theme-plugin';
const STORAGE_KEY = 'zyfun-theme-css';
const THEME_MODE_KEY = 'zyfun-theme-mode'; // 'plugin' | 'builtin'

// 应用插件主题
const applyPluginTheme = (css: string) => {
  const oldStyle = document.getElementById(PLUGIN_STYLE_ID);
  if (oldStyle) oldStyle.remove();

  const styleEl = document.createElement('style');
  styleEl.id = PLUGIN_STYLE_ID;
  styleEl.textContent = css;
  document.head.appendChild(styleEl);

  localStorage.setItem(STORAGE_KEY, css);
  localStorage.setItem(THEME_MODE_KEY, 'plugin');
  console.log('🎨 插件主题已应用');
};

// 移除插件主题（恢复内置）
const removePluginTheme = () => {
  const oldStyle = document.getElementById(PLUGIN_STYLE_ID);
  if (oldStyle) oldStyle.remove();
  localStorage.removeItem(STORAGE_KEY);
  localStorage.setItem(THEME_MODE_KEY, 'builtin');
  console.log('↩️ 已恢复内置主题');
};

// 启动时加载保存的主题
const loadSavedTheme = () => {
  const mode = localStorage.getItem(THEME_MODE_KEY);
  const savedCss = localStorage.getItem(STORAGE_KEY);
  if (mode === 'plugin' && savedCss) {
    requestAnimationFrame(() => applyPluginTheme(savedCss));
  } else {
    removePluginTheme();
  }
};

// 监听插件消息
const handlePluginMessage = (event: MessageEvent) => {
  const data = event.data;
  if (!data || typeof data !== 'object') return;

  if (data.type === 'zyfun-theme-apply' && data.css) {
    applyPluginTheme(data.css);
    return;
  }

  if (data.type === 'zyfun-theme-reset') {
    removePluginTheme();
    return;
  }

  if (data.type === 'zyfun-theme-request') {
    const savedCss = localStorage.getItem(STORAGE_KEY);
    if (savedCss && event.source) {
      event.source.postMessage({
        type: 'zyfun-theme-response',
        css: savedCss
      }, event.origin);
    }
  }
};

// ============================================================
//  原有 watch（检测主题切换，自动移除插件样式）
// ============================================================

watch(
  () => ({
    theme: storeSetting.theme,
    lang: storeSetting.lang,
    debug: storeSetting.debug,
  }),
  (val) => {
    // 如果用户切换了内置主题，且当前是插件模式，则移除插件样式
    if (val.theme !== setupConf.value?.theme) {
      const mode = localStorage.getItem(THEME_MODE_KEY);
      if (mode === 'plugin') {
        removePluginTheme();
      }
      storeSetting.changePreferredTheme();
    }

    if (val.lang !== setupConf.value?.lang) storeSetting.changePreferredLang();
    if (val.debug !== setupConf.value?.debug) debugMode(val.debug);

    for (const key in val) {
      setupConf.value[key] = val[key];
    }
  },
  { deep: true },
);

watch(
  () => storeSetting.displayTheme,
  () => {
    if (storeSetting.theme === THEME.SYSTEM) {
      storeSetting.changePreferredTheme();
    }
  },
);

// ============================================================
//  生命周期
// ============================================================

onMounted(() => {
  setup();
  loadSavedTheme();
  window.addEventListener('message', handlePluginMessage);
});

onUnmounted(() => {
  window.removeEventListener('message', handlePluginMessage);
});

// ============================================================
//  原有 setup 函数
// ============================================================

const setup = () => {
  syncStore();
};

const syncStore = async () => {
  const resp = await fetchSetup();
  setupConf.value = resp;

  const { barrage, bossKey, debug, disclaimer, lang, player, theme, timeout } = resp;

  active.value.disclaimer = !disclaimer;

  storeSetting.updateConfig({ bossKey, debug, lang, theme, timeout: timeout || 5000 });
  storePlayer.updateConfig({ barrage, player });

  if (debug) debugMode(debug);
};

const debugMode = (type: boolean) => {
  if (type) {
    startVitals();
    startOSpy();
  } else {
    stopVitals();
    stopOSpy();
  }
};
</script>
