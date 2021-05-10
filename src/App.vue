<template>
  <pre>{{ isOnline }}</pre>
  <pre>{{ currentRate }}</pre>
  <pre>{{ history }}</pre>
</template>

<script>
import { reactive, ref } from "vue";

export default {
  name: "App",

  setup() {
    const ws = reactive({});
    const history = reactive([]);
    const isOnline = ref(false);

    return {
      // Конечно, в реальном приложении мы не должны хранить
      // любые чувствительные данные на фронте (ключи, токены, ...).
      // В бою, мы бы проксировали запросы через собственный сервер
      // или выдавали ключ к API только аутентифицированным клиентам.
      API_KEY: process?.env?.VUE_APP_API_KEY,
      API_URL: process?.env?.VUE_APP_API_URL,
      ws,
      history,
      isOnline
    };
  },

  created() {
    if (this.API_URL && this.API_KEY) {
      try {
        // Инициализируем WS и регистрируем обработчики
        this.ws = new WebSocket(this.API_URL);

        this.ws.onopen = () => {
          console.log("WS was opened");
          // TODO: add symbols
          this.ws.send(`{"userKey":"${this.API_KEY}", "symbol":"GBPUSD"}`);
          this.isOnline = true;
        };

        this.ws.onmessage = msg => {
          console.log("WS got a message", msg?.data);
          this.history.push(msg?.data);
          if (this.history?.length > 10) {
            // храним только последние 10 сообщений
            this.history.shift();
          }
        };

        this.ws.onclose = () => {
          console.log("WS was closed");
          // TODO: add reconnect
          this.isOnline = false;
        };

        this.ws.onerror = err => {
          console.log("WS caught an Error", err);
          this.isOnline = false;
        };
      } catch (e) {
        console.error(e);
      }
    } else {
      console.warn("API_URL and/or API_KEY couldn't be found.");
    }
  },

  computed: {
    currentRate() {
      return this?.history[this?.history?.length - 1] || {};
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
