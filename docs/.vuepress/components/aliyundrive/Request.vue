<script lang="ts" setup>
import {
  NButton,
  NSpace,
  NAlert,
  NImage,
  useMessage,
  NCheckbox,
  NInput,
} from "naive-ui"
import { ref } from "vue"
import { api } from "../api"

const alist_redirect_uri = "https://alistgo.com/tool/aliyundrive/callback"
const app_id = "b8c990e60b18446eb07f5dca30398e8a"

function goGet() {
  const url = new URL("https://open.aliyundrive.com/oauth/authorize")
  url.searchParams.set("client_id", useMyOwn.value ? client_id.value : app_id)
  url.searchParams.set("redirect_uri", alist_redirect_uri)
  url.searchParams.set("scope", "user:base,file:all:read,file:all:write")
  url.searchParams.set("response_type", "code")
  url.searchParams.set(
    "state",
    window.btoa(
      useMyOwn.value ? `${client_id.value}::${client_secret.value}` : "::"
    )
  )
  url.searchParams.set("relogin", "true")
  window.open(url.toString(), "_blank")
}

interface QrCodeResp {
  qrCodeUrl: string
  sid: string
  error: string
}

const qrcode = ref<QrCodeResp>()

const gettingQrCode = ref(false)
const client_id = ref("")
const client_secret = ref("")
const useMyOwn = ref(false)

async function getQrCode() {
  try {
    gettingQrCode.value = true
    const resp = await fetch(api() + "/alist/ali_open/qr", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(
        useMyOwn.value
          ? {
              client_id: client_id.value,
              client_secret: client_secret.value,
            }
          : {}
      ),
    })
    const res: QrCodeResp = await resp.json()
    qrcode.value = res
  } catch (e) {
    console.log(e)
    qrcode.value = {
      error: "error",
      qrCodeUrl: "",
      sid: "",
    }
  } finally {
    gettingQrCode.value = false
  }
}

interface ScanStatus {
  status: string
  authCode: string
}

const scanStatus = ref<ScanStatus>()
const gettingScanStatus = ref(false)

const message = useMessage()

async function haveScan() {
  try {
    gettingScanStatus.value = true
    const resp = await fetch(
      `${api()}/proxy/https://open.aliyundrive.com/oauth/qrcode/${
        qrcode.value?.sid
      }/status`
    )
    const res: ScanStatus = await resp.json()
    scanStatus.value = res
    if (res.status === "LoginSuccess") {
      const url = new URL(alist_redirect_uri)
      url.searchParams.set("code", res.authCode)
      window.open(url.toString(), "_blank")
    } else {
      message.warning(res.status)
    }
  } catch (e) {
    console.log(e)
    message.error(String(e))
  } finally {
    gettingScanStatus.value = false
  }
}
</script>

<template>
  <NSpace vertical size="large">
    <NCheckbox v-model:checked="useMyOwn" size="large"
      >使用自己的客户端信息</NCheckbox
    >
    <template v-if="useMyOwn">
      <NInput size="large" v-model:value="client_id" placeholder="client_id" />
      <NInput
        size="large"
        v-model:value="client_secret"
        placeholder="client_secret"
      />
    </template>
    <NButton size="large" type="primary" @click="goGet" block
      >前往登录</NButton
    >
    <NButton
      size="large"
      type="info"
      @click="getQrCode"
      block
      :loading="gettingQrCode"
      :disabled="!!qrcode"
      >扫描二维码
    </NButton>
    <NAlert title="Error" type="error" v-if="qrcode?.error">
      {{ qrcode.error }}
    </NAlert>
    <NSpace v-if="qrcode?.qrCodeUrl" vertical>
      <NSpace justify="center">
        <NImage width="300" :src="qrcode.qrCodeUrl" />
      </NSpace>
      <NAlert title="扫描二维码" type="info"
        >使用 AliyunDrive APP 扫描，然后点击下方按钮</NAlert
      >
      <NButton
        size="large"
        @click="haveScan"
        type="info"
        block
        :loading="gettingScanStatus"
        >已扫码</NButton
      >
    </NSpace>
    <NAlert
      title="Warning"
      type="warning"
      v-if="scanStatus && scanStatus.status !== 'LoginSuccess'"
    >
      {{ scanStatus.status }}
    </NAlert>
  </NSpace>
</template>

<style scoped>
h4 {
  margin: 0;
}
</style>
