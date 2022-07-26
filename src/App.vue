<script setup lang="ts">
import Index from './components/Index.vue'
import { ElConfigProvider } from 'element-plus';
import zhCn from 'element-plus/lib/locale/lang/zh-cn';
</script>

<template>
  <el-config-provider :locale="zhCn">
    <Index />
  </el-config-provider>
</template>

<style>
@import './assets/base.css';
</style>
