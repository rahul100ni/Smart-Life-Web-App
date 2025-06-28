<template>
  <div class="app-container">
    <!-- Premium Login Screen -->
    <div v-if="!loginState" class="login-container">
      <div class="login-content">
        <!-- Brand Identity -->
        <div class="brand-section">
          <div class="brand-logo">
            <div class="logo-glow">
              <svg width="48" height="48" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path d="M9 22C9 22 3 18 3 12V5L12 2L21 5V12C21 18 15 22 15 22" stroke="url(#gradient)" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                <path d="M12 8V16M8 12H16" stroke="url(#gradient)" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                <defs>
                  <linearGradient id="gradient" x1="0%" y1="0%" x2="100%" y2="100%">
                    <stop offset="0%" style="stop-color:#00D4FF"/>
                    <stop offset="100%" style="stop-color:#5B73FF"/>
                  </linearGradient>
                </defs>
              </svg>
            </div>
          </div>
          <h1 class="brand-title">SMART LIFE</h1>
          <p class="brand-subtitle">Intelligent Living, Simplified</p>
        </div>

        <!-- Login Form -->
        <div class="login-form-container">
          <h2 class="form-title">Welcome Back</h2>
          
          <form @submit.prevent="login" class="login-form">
            <div class="input-group">
              <div class="input-wrapper">
                <input 
                  v-model="loginForm.username"
                  type="email"
                  placeholder="Email address"
                  class="premium-input"
                  required
                />
                <div class="input-border"></div>
              </div>
            </div>

            <div class="input-group">
              <div class="input-wrapper">
                <input 
                  v-model="loginForm.password"
                  type="password"
                  placeholder="Password"
                  class="premium-input"
                  required
                />
                <div class="input-border"></div>
              </div>
            </div>

            <button 
              type="submit" 
              class="login-button"
              :disabled="isLoading"
            >
              <span v-if="!isLoading" class="button-content">
                <span>Sign In</span>
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path d="M5 12H19M19 12L12 5M19 12L12 19" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </span>
              <span v-else class="loading-content">
                <div class="loading-spinner"></div>
                <span>Authenticating...</span>
              </span>
            </button>
          </form>

          <div class="form-footer">
            <p class="security-note">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none">
                <rect x="3" y="11" width="18" height="11" rx="2" ry="2" stroke="currentColor" stroke-width="2"/>
                <circle cx="12" cy="16" r="1" fill="currentColor"/>
                <path d="M7 11V7A5 5 0 0 1 17 7V11" stroke="currentColor" stroke-width="2"/>
              </svg>
              End-to-end encrypted connection
            </p>
          </div>
        </div>
      </div>

      <!-- Animated Background -->
      <div class="background-effects">
        <div class="neural-network">
          <div class="node node-1"></div>
          <div class="node node-2"></div>
          <div class="node node-3"></div>
          <div class="node node-4"></div>
          <div class="node node-5"></div>
          <div class="connection connection-1"></div>
          <div class="connection connection-2"></div>
          <div class="connection connection-3"></div>
        </div>
        <div class="gradient-orb orb-1"></div>
        <div class="gradient-orb orb-2"></div>
        <div class="grid-overlay"></div>
      </div>
    </div>

    <!-- Main App (when logged in) -->
    <div v-else class="main-app">
      <div class="app-header">
        <div class="header-content">
          <h1 class="app-title">Smart Life Dashboard</h1>
          <div class="header-actions">
            <el-button type="default" @click="refreshDevices()" class="header-btn">
              <i class="material-icons-round">refresh</i>
              Refresh
            </el-button>
            <el-button type="default" @click="logout()" class="header-btn logout-btn">
              <i class="material-icons-round">logout</i>
              Logout
            </el-button>
          </div>
        </div>
      </div>
      
      <div class="devices-container">
        <div v-for="device in devicesSorted" :key="device.id">
          <el-card class="device" :style="device.data.online === false ? 'filter: opacity(0.65) grayscale(1);' : ''">
            <el-tooltip effect="light" :content="device.type" :offset="-20"
              :visible-arrow="false">
              <el-avatar :src="`/device_icons/${device.type}.png`" shape="square">
                <img src="/device_icons/default.png"/>
              </el-avatar>
            </el-tooltip>
            <span class="device-name">{{ device.name }}</span>
            <template v-if="device.type === 'scene'">
              <el-button type="default" circle size="large"
                class="trigger"
                @click="triggerScene(device);"
              ><i class="material-icons-round">play_arrow</i></el-button>
            </template>
            <template v-else>
              <el-button type="default" circle size="large"
                :class="device.data.state ? 'state-on' : 'state-off'"
                :disabled="!device.data.online"
                @click="toggleDevice(device);"
              ><i class="material-icons-round">{{ device.data.online ? 'power_settings_new' : 'cloud_off' }}</i></el-button>
            </template>
          </el-card>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Home'
}
</script>

<script setup="" >
/* eslint-disable no-unused-vars */
import { ref, reactive, computed, onMounted } from 'vue'
import { ElMessage } from "element-plus"

import tuya from '@/libs/tuya'

const homeAssistantClient = new tuya.HomeAssistantClient(
  JSON.parse(localStorage.getItem('session'))
)

const loginState = ref(false)
const devices = ref([])
const isLoading = ref(false)

const devicesSorted = computed(() => {
  const order = { true: 0, undefined: 1, false: 2 }
  return devices.value.slice().sort((d1, d2) =>
    order[d1.data.online] > order[d2.data.online] ? 1 : -1
  )
})

const loginForm = ref({ username: '', password: '' })

onMounted(async () => {
  // TODO handle expired session
  loginState.value = !!homeAssistantClient.getSession()
  if (!loginState.value) {
    localStorage.clear()
  }
  devices.value = JSON.parse(localStorage.getItem('devices')) || []
})

const login = async () => {
  if (!loginForm.value.username || !loginForm.value.password) {
    ElMessage.warning('Please enter both email and password')
    return
  }
  
  isLoading.value = true
  try {
    await homeAssistantClient.login(
      loginForm.value.username,
      loginForm.value.password
    )
    localStorage.setItem('session', JSON.stringify(homeAssistantClient.getSession()))
    loginState.value = true
    loginForm.value = { username: '', password: '' }
    refreshDevices()
    ElMessage.success('Successfully logged in!')
  } catch (err) {
    ElMessage.error(`Login failed: ${err.message || err}`)
  } finally {
    isLoading.value = false
  }
}

const logout = () => {
  homeAssistantClient.dropSession()
  localStorage.clear()
  loginState.value = false
  loginForm.value = { username: '', password: '' }
  devices.value = []
  ElMessage.success('Successfully logged out')
}

const refreshDevices = async () => {
  // TODO handle expired session
  try {
    const discoveryResponse = await homeAssistantClient.deviceDiscovery()
    const discoveryDevices = discoveryResponse.payload.devices || []
    devices.value = discoveryDevices
    localStorage.setItem('devices', JSON.stringify(discoveryDevices))
    ElMessage.success(`Found ${discoveryDevices.length} devices`)
  } catch (err) {
    ElMessage.error(`Device discovery failed: ${err.message || err}`)
  }
}

const toggleDevice = async (device) => {
  // TODO handle expired session
  // TODO change icon to el-icon-loading
  try {
    const newState = !device.data.state
    await homeAssistantClient.deviceControl(device.id, 'turnOnOff', newState)
    device.data.state = newState
  } catch (err) {
    ElMessage.error(`Device control failed: ${err.message || err}`)
  }
}

const triggerScene = async (device) => {
  // TODO handle expired session
  // TODO change icon to el-icon-loading
  try {
    await homeAssistantClient.deviceControl(device.id, 'turnOnOff', true)
    ElMessage.success(`Scene "${device.name}" triggered`)
  } catch (err) {
    ElMessage.error(`Scene trigger failed: ${err.message || err}`)
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

* {
  box-sizing: border-box;
}

.app-container {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  min-height: 100vh;
}

/* Premium Login Screen */
.login-container {
  min-height: 100vh;
  background: radial-gradient(ellipse at top, #0F0F23 0%, #000000 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  position: relative;
  overflow: hidden;
}

.login-content {
  position: relative;
  z-index: 10;
  width: 100%;
  max-width: 420px;
  animation: fadeInUp 0.8s ease-out;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Brand Section */
.brand-section {
  text-align: center;
  margin-bottom: 3rem;
}

.brand-logo {
  margin-bottom: 1.5rem;
}

.logo-glow {
  width: 80px;
  height: 80px;
  margin: 0 auto;
  background: radial-gradient(circle, rgba(0, 212, 255, 0.2) 0%, transparent 70%);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  animation: pulse 3s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(0, 212, 255, 0.4);
  }
  50% {
    transform: scale(1.05);
    box-shadow: 0 0 0 20px rgba(0, 212, 255, 0);
  }
}

.brand-title {
  font-size: 2.5rem;
  font-weight: 700;
  letter-spacing: 0.1em;
  background: linear-gradient(135deg, #00D4FF 0%, #5B73FF 50%, #FFFFFF 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin: 0 0 0.5rem 0;
  text-transform: uppercase;
}

.brand-subtitle {
  color: rgba(255, 255, 255, 0.6);
  font-size: 1rem;
  font-weight: 400;
  margin: 0;
  letter-spacing: 0.05em;
}

/* Login Form */
.login-form-container {
  background: rgba(255, 255, 255, 0.02);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 24px;
  padding: 2.5rem;
  box-shadow: 
    0 20px 40px rgba(0, 0, 0, 0.4),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.form-title {
  color: #FFFFFF;
  font-size: 1.5rem;
  font-weight: 600;
  margin: 0 0 2rem 0;
  text-align: center;
  letter-spacing: -0.02em;
}

.login-form {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.input-group {
  position: relative;
}

.input-wrapper {
  position: relative;
}

.premium-input {
  width: 100%;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 16px;
  padding: 1rem 1.25rem;
  font-size: 1rem;
  color: #FFFFFF;
  font-weight: 400;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  outline: none;
  font-family: inherit;
}

.premium-input::placeholder {
  color: rgba(255, 255, 255, 0.4);
  font-weight: 400;
}

.premium-input:focus {
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(0, 212, 255, 0.5);
  box-shadow: 
    0 0 0 3px rgba(0, 212, 255, 0.1),
    0 8px 32px rgba(0, 212, 255, 0.15);
  transform: translateY(-1px);
}

.input-border {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, #00D4FF, #5B73FF);
  border-radius: 1px;
  transform: scaleX(0);
  transition: transform 0.3s ease;
}

.premium-input:focus + .input-border {
  transform: scaleX(1);
}

.login-button {
  background: linear-gradient(135deg, #00D4FF 0%, #5B73FF 100%);
  border: none;
  border-radius: 16px;
  padding: 1rem 2rem;
  font-size: 1rem;
  font-weight: 600;
  color: #FFFFFF;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
  margin-top: 0.5rem;
  font-family: inherit;
  letter-spacing: 0.02em;
}

.login-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 
    0 12px 40px rgba(0, 212, 255, 0.3),
    0 0 0 1px rgba(255, 255, 255, 0.1);
}

.login-button:active:not(:disabled) {
  transform: translateY(-1px);
}

.login-button:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.button-content {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.loading-content {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
}

.loading-spinner {
  width: 20px;
  height: 20px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top: 2px solid #FFFFFF;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.form-footer {
  margin-top: 2rem;
  text-align: center;
}

.security-note {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  color: rgba(255, 255, 255, 0.5);
  font-size: 0.875rem;
  margin: 0;
  font-weight: 400;
}

/* Background Effects */
.background-effects {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  z-index: 1;
}

.neural-network {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.node {
  position: absolute;
  width: 4px;
  height: 4px;
  background: rgba(0, 212, 255, 0.6);
  border-radius: 50%;
  animation: nodeGlow 4s ease-in-out infinite;
}

.node-1 { top: 20%; left: 15%; animation-delay: 0s; }
.node-2 { top: 30%; right: 20%; animation-delay: 1s; }
.node-3 { bottom: 40%; left: 25%; animation-delay: 2s; }
.node-4 { bottom: 25%; right: 15%; animation-delay: 3s; }
.node-5 { top: 60%; left: 50%; animation-delay: 1.5s; }

@keyframes nodeGlow {
  0%, 100% {
    opacity: 0.3;
    transform: scale(1);
  }
  50% {
    opacity: 1;
    transform: scale(1.5);
  }
}

.connection {
  position: absolute;
  height: 1px;
  background: linear-gradient(90deg, transparent, rgba(0, 212, 255, 0.3), transparent);
  animation: connectionFlow 6s ease-in-out infinite;
}

.connection-1 {
  top: 25%;
  left: 15%;
  width: 200px;
  transform: rotate(25deg);
  animation-delay: 0s;
}

.connection-2 {
  bottom: 35%;
  right: 20%;
  width: 150px;
  transform: rotate(-45deg);
  animation-delay: 2s;
}

.connection-3 {
  top: 55%;
  left: 30%;
  width: 180px;
  transform: rotate(15deg);
  animation-delay: 4s;
}

@keyframes connectionFlow {
  0%, 100% {
    opacity: 0;
  }
  50% {
    opacity: 0.6;
  }
}

.gradient-orb {
  position: absolute;
  border-radius: 50%;
  filter: blur(60px);
  animation: orbFloat 8s ease-in-out infinite;
}

.orb-1 {
  width: 300px;
  height: 300px;
  background: radial-gradient(circle, rgba(0, 212, 255, 0.15) 0%, transparent 70%);
  top: -150px;
  right: -150px;
  animation-delay: 0s;
}

.orb-2 {
  width: 250px;
  height: 250px;
  background: radial-gradient(circle, rgba(91, 115, 255, 0.1) 0%, transparent 70%);
  bottom: -125px;
  left: -125px;
  animation-delay: 4s;
}

@keyframes orbFloat {
  0%, 100% {
    transform: translate(0, 0) scale(1);
  }
  50% {
    transform: translate(20px, -20px) scale(1.1);
  }
}

.grid-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: 
    linear-gradient(rgba(255, 255, 255, 0.02) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.02) 1px, transparent 1px);
  background-size: 50px 50px;
  animation: gridShift 20s linear infinite;
}

@keyframes gridShift {
  0% {
    transform: translate(0, 0);
  }
  100% {
    transform: translate(50px, 50px);
  }
}

/* Main App Styles (keeping existing) */
.main-app {
  min-height: 100vh;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
}

.app-header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(20px);
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  padding: 1.5rem 0;
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-title {
  font-size: 1.8rem;
  font-weight: 700;
  color: #2d3748;
  margin: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.header-actions {
  display: flex;
  gap: 1rem;
}

.header-btn {
  border-radius: 10px;
  padding: 0.75rem 1.5rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.3s ease;
}

.header-btn:hover {
  transform: translateY(-1px);
}

.logout-btn {
  background: #fee;
  border-color: #fed7d7;
  color: #e53e3e;
}

.logout-btn:hover {
  background: #fed7d7;
}

.devices-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.el-card.device {
  margin-bottom: 1rem;
  border-radius: 16px;
  border: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  transition: all 0.3s ease;
}

.el-card.device:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
}

.el-card.device :deep(.el-card__body) {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  padding: 1.5rem;
}

.el-card.device :deep(.el-card__body :last-child) {
  margin-left: auto;
}

.device-name {
  font-weight: 500;
  color: #2d3748;
  font-size: 1.1rem;
}

.el-button.state-on:enabled {
  color: #f9f9f9;
  background-color: #48bb78;
  border-color: #48bb78;
}

.el-button.state-off:enabled {
  color: #718096;
  background-color: #f7fafc;
  border-color: #e2e8f0;
}

.el-button.trigger:enabled {
  color: #f9f9f9;
  background-color: #667eea;
  border-color: #667eea;
}

.el-button.el-button--large {
  padding: 12px;
  font-size: 20px;
  line-height: 0px;
  border-radius: 12px;
  transition: all 0.3s ease;
}

.el-button.el-button--large:hover {
  transform: scale(1.05);
}

.el-avatar {
  background: transparent;
  margin-right: 1rem;
}

/* Responsive Design */
@media (max-width: 768px) {
  .login-content {
    max-width: 100%;
    padding: 0 1rem;
  }
  
  .login-form-container {
    padding: 2rem;
  }
  
  .brand-title {
    font-size: 2rem;
  }
  
  .header-content {
    flex-direction: column;
    gap: 1rem;
    text-align: center;
  }
  
  .header-actions {
    justify-content: center;
  }
  
  .devices-container {
    padding: 1rem;
  }
}

@media (max-width: 480px) {
  .login-form-container {
    padding: 1.5rem;
  }
  
  .brand-title {
    font-size: 1.75rem;
  }
}
</style>