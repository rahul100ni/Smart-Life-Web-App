<template>
  <div class="app-container">
    <!-- Login Screen -->
    <div v-if="!loginState" class="login-container">
      <div class="login-card">
        <div class="login-header">
          <div class="logo-container">
            <div class="logo-icon">
              <i class="material-icons-round">home</i>
            </div>
          </div>
          <h1 class="login-title">Smart Life</h1>
          <p class="login-subtitle">Control your smart devices from anywhere</p>
        </div>
        
        <el-form :model="loginForm" class="login-form" @submit.prevent="login">
          <div class="form-group">
            <label class="form-label">Email Address</label>
            <el-input 
              v-model="loginForm.username" 
              type="email"
              placeholder="Enter your email"
              size="large"
              class="form-input"
              :prefix-icon="'User'"
            />
          </div>
          
          <div class="form-group">
            <label class="form-label">Password</label>
            <el-input 
              v-model="loginForm.password" 
              type="password"
              placeholder="Enter your password"
              size="large"
              class="form-input"
              :prefix-icon="'Lock'"
              show-password
            />
          </div>
          
          <el-button 
            type="primary" 
            size="large"
            class="login-button"
            @click="login()"
            :loading="isLoading"
          >
            <span v-if="!isLoading">Sign In</span>
            <span v-else>Signing In...</span>
          </el-button>
        </el-form>
        
        <div class="login-footer">
          <p class="footer-text">
            Secure connection to your Smart Life account
          </p>
        </div>
      </div>
      
      <!-- Background decoration -->
      <div class="background-decoration">
        <div class="decoration-circle circle-1"></div>
        <div class="decoration-circle circle-2"></div>
        <div class="decoration-circle circle-3"></div>
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
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

/* Login Screen Styles */
.login-container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  position: relative;
  overflow: hidden;
}

.login-card {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(20px);
  border-radius: 24px;
  padding: 3rem;
  width: 100%;
  max-width: 440px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  z-index: 10;
  animation: slideUp 0.6s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.login-header {
  text-align: center;
  margin-bottom: 2.5rem;
}

.logo-container {
  margin-bottom: 1.5rem;
}

.logo-icon {
  width: 80px;
  height: 80px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto;
  box-shadow: 0 10px 30px rgba(102, 126, 234, 0.3);
}

.logo-icon i {
  font-size: 2.5rem;
  color: white;
}

.login-title {
  font-size: 2.5rem;
  font-weight: 700;
  color: #2d3748;
  margin: 0 0 0.5rem 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.login-subtitle {
  color: #718096;
  font-size: 1.1rem;
  margin: 0;
  font-weight: 400;
}

.login-form {
  margin-bottom: 2rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-label {
  display: block;
  font-weight: 600;
  color: #4a5568;
  margin-bottom: 0.5rem;
  font-size: 0.95rem;
}

.form-input :deep(.el-input__wrapper) {
  border-radius: 12px;
  border: 2px solid #e2e8f0;
  box-shadow: none;
  transition: all 0.3s ease;
  padding: 0.75rem 1rem;
}

.form-input :deep(.el-input__wrapper:hover) {
  border-color: #cbd5e0;
}

.form-input :deep(.el-input__wrapper.is-focus) {
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.form-input :deep(.el-input__inner) {
  font-size: 1rem;
  color: #2d3748;
}

.form-input :deep(.el-input__inner::placeholder) {
  color: #a0aec0;
}

.login-button {
  width: 100%;
  height: 56px;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 600;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
  transition: all 0.3s ease;
  margin-top: 1rem;
}

.login-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 25px rgba(102, 126, 234, 0.3);
}

.login-button:active {
  transform: translateY(0);
}

.login-footer {
  text-align: center;
  padding-top: 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.footer-text {
  color: #718096;
  font-size: 0.9rem;
  margin: 0;
}

/* Background Decoration */
.background-decoration {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  z-index: 1;
}

.decoration-circle {
  position: absolute;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  animation: float 6s ease-in-out infinite;
}

.circle-1 {
  width: 200px;
  height: 200px;
  top: 10%;
  left: 10%;
  animation-delay: 0s;
}

.circle-2 {
  width: 150px;
  height: 150px;
  top: 60%;
  right: 10%;
  animation-delay: 2s;
}

.circle-3 {
  width: 100px;
  height: 100px;
  bottom: 20%;
  left: 20%;
  animation-delay: 4s;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0px) rotate(0deg);
  }
  50% {
    transform: translateY(-20px) rotate(180deg);
  }
}

/* Main App Styles */
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

/* Device Cards (keeping existing styles but with minor improvements) */
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
  .login-card {
    padding: 2rem;
    margin: 1rem;
  }
  
  .login-title {
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
</style>