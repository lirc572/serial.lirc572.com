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
      <v-card>
        <v-card-title class="headline">
          Serial Monitor
        </v-card-title>
        <v-card-text>
          <v-btn
            elevation="2"
            color="primary"
            :disabled="!serialSupported"
            @click="selectPort"
          >
            Select Port
          </v-btn>
          <v-btn
            elevation="2"
            color="success"
            :disabled="!port"
            @click="connect"
          >
            {{ inputStream ? 'Disconnect' : 'Connect' }}
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
              @keyup="inputTextKeyUp"
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
export default {
  data () {
    return {
      port: null,
      readableStreamClosed: null,
      writableStreamClosed: null,
      inputStream: null,
      outputStream: null,
      reader: null,
      consoleText: '',
      inputText: '',
      serialSupported: null
    }
  },
  created () {
    let serialSupported = 'ha'
    if (process.client) {
      // running on client
      serialSupported = 'serial' in navigator
    }
    this.serialSupported = serialSupported
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
    async connect () {
      if (!this.inputStream) {
        await this.openPort()
      } else {
        if (this.reader) {
          this.reader.cancel()
          await this.readableStreamClosed.catch(() => { /* Ignore the error */ })
          this.reader = null
          this.inputStream = null
          this.readableStreamClosed = null
        }
        if (this.outputStream) {
          await this.outputStream.getWriter().close()
          await this.writableStreamClosed
          this.outputStream = null
          this.writableStreamClosed = null
        }
        await this.port.close()
        this.port = null
        this.consoleText = ''
        this.inputText = ''
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
      this.readableStreamClosed = this.port.readable.pipeTo(decoder.writable)
      this.inputStream = decoder.readable
      this.reader = this.inputStream.getReader()
    },
    createOutputStream () {
      // eslint-disable-next-line no-undef
      const encoder = new TextEncoderStream()
      this.writableStreamClosed = encoder.readable.pipeTo(this.port.writable)
      this.outputStream = encoder.writable
    },
    writeToOutputStream () {
      const writer = this.outputStream.getWriter()
      const lines = this.inputText.split('\n')
      lines.forEach((line) => {
        writer.write(line + '\n')
        this.updateConsoleText(line + '\n')
      })
      this.inputText = ''
      writer.releaseLock()
    },
    inputTextKeyUp (event) {
      if (event.keyCode === 13) {
        this.writeToOutputStream()
      }
    },
    async readLoop () {
      while (true) {
        const { value, done } = await this.reader.read()
        if (value) {
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
