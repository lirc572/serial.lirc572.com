<template>
  <v-row justify="center" align="center">
    <v-col cols="12" sm="8" md="6">
      <v-alert
        v-show="!serialSupported"
        border="left"
        type="error"
        elevation="2"
      >
        Web Serial API is not supported on your browser! Please use the latest version of Chrome!
      </v-alert>
      <div class="text-center">
        <logo />
        <vuetify-logo />
      </div>
      <v-card>
        <v-card-title class="headline">
          Serial Monitor
        </v-card-title>
        <v-card-text>
          <v-btn
            elevation="2"
            color="primary"
            @click="selectPort"
          >
            Select Port
          </v-btn>
          <v-btn
            elevation="2"
            color="success"
            :disabled="!port"
            @click="openPort"
          >
            Connect
          </v-btn>
          <hr class="my-3">
          <v-textarea
            v-show="reader"
            id="console-textarea"
            outlined
            readonly
            name="console-textarea"
            label="Console"
            :value="consoleText"
          />
          <v-row
            v-show="reader"
            id="send-row"
            justify="center"
            align="baseline"
          >
            <v-text-field
              v-model="inputText"
              label="Send Text"
              clearable
            />
            <v-btn
              elevation="2"
              color="cyan"
              @click="writeToOutputStream"
            >
              Send
            </v-btn>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import Logo from '~/components/Logo.vue'
import VuetifyLogo from '~/components/VuetifyLogo.vue'

export default {
  components: {
    Logo,
    VuetifyLogo
  },
  asyncData () {
    return {
      serialSupported: process.browser ? ('serial' in navigator) : true
    }
  },
  data () {
    return {
      port: null,
      inputStream: null,
      reader: null,
      outputStream: null,
      consoleText: '',
      inputText: ''
    }
  },
  methods: {
    updateConsoleText (text) {
      this.consoleText = this.consoleText.slice(0, -1) + text + '\n'
    },
    async selectPort () {
      try {
        this.port = await navigator.serial.requestPort()
      } catch (error) {
        // eslint-disable-next-line no-console
        console.log(error)
      }
    },
    async openPort () {
      try {
        await this.port.open({ baudRate: 115200 })
        this.createInputStream()
        this.createOutputStream()
        this.readLoop()
      } catch (error) {
        // eslint-disable-next-line no-console
        console.log(error)
      }
    },
    createInputStream () {
      // eslint-disable-next-line no-undef
      const decoder = new TextDecoderStream()
      this.port.readable.pipeTo(decoder.writable)
      this.inputStream = decoder.readable
      this.reader = this.inputStream.getReader()
    },
    createOutputStream () {
      // eslint-disable-next-line no-undef
      const encoder = new TextEncoderStream()
      encoder.readable.pipeTo(this.port.writable)
      this.outputStream = encoder.writable
    },
    writeToOutputStream () {
      const lines = this.inputText.split('\n')
      const writer = this.outputStream.getWriter()
      lines.forEach((line) => {
        // eslint-disable-next-line no-console
        console.log('[SEND]', line)
        writer.write(line + '\n')
        this.updateConsoleText(line + '\n')
      })
      this.inputText = ''
      writer.releaseLock()
    },
    async readLoop () {
      while (true) {
        const { value, done } = await this.reader.read()
        if (value) {
          // eslint-disable-next-line no-console
          console.log(value)
          this.updateConsoleText(value)
          this.consoleScrollToBottom()
        }
        if (done) {
          // eslint-disable-next-line no-console
          console.log('[readLoop] DONE', done)
          this.reader.releaseLock()
          break
        }
      }
    },
    consoleScrollToBottom () {
      const consoleElement = document.getElementById('console-textarea')
      // eslint-disable-next-line no-console
      console.log('Scrolling to', consoleElement.scrollHeight)
      consoleElement.scrollTop = consoleElement.scrollHeight
    }
  }
}
</script>
<style scoped>
#send-row {
  margin-left: 0px;
  margin-right: 0px;
}
</style>
