<template>
  <div class="wrapper">
    <v-sheet width="360" class="mx-auto login-block">
      <h1 class="login-block-h1">Admin Panel</h1>
      <p v-if="step === 'setup'" class="login-help">
        Scan this QR with <b>Google Authenticator</b> (or type the secret), then enter the 6-digit code. Codes change every 30 seconds.
      </p>
      <p v-else-if="step === 'otp'" class="login-help">
        Enter the 6-digit code from Google Authenticator.
      </p>
      <v-form fast-fail @submit.prevent="login()">
        <v-text-field
          v-if="step === 'credentials'"
          v-model="user"
          label="Login"
        ></v-text-field>

        <v-text-field
          v-if="step === 'credentials'"
          v-model="pass"
          label="Password"
          type="password"
        ></v-text-field>

        <div v-if="step === 'setup'" class="setup-block">
          <img v-if="qrDataUrl" :src="qrDataUrl" alt="Google Authenticator QR" class="qr-img" />
          <div class="secret-label">Secret (if you cannot scan)</div>
          <div class="secret-key">{{ secret }}</div>
        </div>

        <v-text-field
          v-if="step === 'setup' || step === 'otp'"
          v-model="code"
          label="6-digit code"
          autocomplete="one-time-code"
          inputmode="numeric"
        ></v-text-field>

        <v-btn block class="mt-2" variant="tonal" type="submit" @click="login()">
          {{ submitLabel }}
        </v-btn>
      </v-form>
    </v-sheet>
  </div>
  <v-alert
    class="alert-block"
    v-if="alert"
    color="pink"
    dark
    border="top"
    icon="mdi-home"
    transition="scale-transition"
  >
    {{ alertText }}
  </v-alert>
</template>

<script setup>
import { computed, ref } from 'vue'
import axios from 'axios'
import qrcode from 'qrcode-generator'
import localConfig from "@/local_config"
import { useRouter } from 'vue-router'
import { findErrMessage } from "@/plugins/helpers"
const apiKey = localConfig.api
const user = ref("")
const pass = ref("")
const code = ref("")
const secret = ref("")
const otpauthUrl = ref("")
const qrDataUrl = ref("")
const step = ref("credentials")
const alert = ref(false)
const alertText = ref('')
const router = useRouter()

const submitLabel = computed(() => {
  if (step.value === "setup") return "Verify and enable 2FA"
  if (step.value === "otp") return "Sign in"
  return "Continue"
})

const makeQr = (url) => {
  const qr = qrcode(0, "M")
  qr.addData(url)
  qr.make()
  qrDataUrl.value = qr.createDataURL(6, 4)
}

const finishLogin = (data) => {
  localStorage.setItem("jwt_token", data.access_token)
  router.push({path: '/page/dashboard'})
}

const login = async () => {
  try {
    const response = await axios.post(`${apiKey}login/`, {
      username: user.value,
      password: pass.value,
      otp_token: code.value
    });
    const data = response.data || {}
    if (data.status === true && data.access_token) {
      finishLogin(data)
      return
    }
    if (data.otp_setup_required) {
      step.value = "setup"
      secret.value = data.secret || ""
      otpauthUrl.value = data.otpauth_url || ""
      code.value = ""
      if (otpauthUrl.value) makeQr(otpauthUrl.value)
      return
    }
    if (data.otp_required) {
      step.value = "otp"
      code.value = ""
      return
    }
    showAlert({ response: { data } })
  } catch (error) {
    const data = error?.response?.data || {}
    if (data.otp_setup_required) {
      step.value = "setup"
      secret.value = data.secret || secret.value
      otpauthUrl.value = data.otpauth_url || otpauthUrl.value
      if (otpauthUrl.value) makeQr(otpauthUrl.value)
    } else if (data.otp_required) {
      step.value = "otp"
    }
    showAlert(error)
  }
}

const showAlert = (err) => {
  const alertMessage = findErrMessage(err)
  if(alertMessage) {
    alert.value = true
    alertText.value = alertMessage
    setTimeout(() => {
      alert.value = false
      alertText.value = ''
    }, 3000)
  }
}

</script>

<style>
.wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #38485C;
}
input {
    border: 1px solid #CCC;
    padding: 5px;
}
label {
    display: block;
    width: 150px;
}
.alert-block {
  position: fixed !important;
  bottom: 0 !important;
  right: 0 !important;
  width: 520px !important;
  z-index: 22 !important;
}
.login-block {
  padding: 40px;
}
.login-block-h1 {
  font-size: 25px;
  font-weight: bold;
  padding-bottom: 10px;
  text-align: center;
}
.login-help {
  font-size: 13px;
  color: #555;
  margin-bottom: 16px;
  line-height: 1.4;
  text-align: center;
}
.setup-block {
  text-align: center;
  margin-bottom: 12px;
}
.qr-img {
  width: 180px;
  height: 180px;
  background: #fff;
  padding: 8px;
  border-radius: 4px;
}
.secret-label {
  font-size: 12px;
  color: #777;
  margin-top: 10px;
}
.secret-key {
  font-family: monospace;
  font-size: 13px;
  word-break: break-all;
  margin: 6px 0 12px;
}
</style>
