<template>
  <h1>Курсы валют</h1>

  <h2>
    <span>Актуальный курс</span> |
    <span
      :class="{ __online: isOnline }"
      class="connection"
      v-text="isOnline ? 'онлайн' : 'оффлайн'"
    />
  </h2>

  <div class="rates-container">
    <div class="row">
      <div class="col">Пара</div>
      <div class="col">Продажа</div>
      <div class="col">Покупка</div>
    </div>
    <div v-for="rate of currentRateList" :key="rate.symbol" class="row">
      <div class="col" v-text="rate.symbol" />
      <div class="col" v-text="rate.ask" />
      <div class="col" v-text="rate.bid" />
    </div>
  </div>

  <h2>Недавняя история</h2>
  <code v-for="(symbolHistory, symbol) of history" :key="symbol">
    <h3>{{ symbol }}</h3>
    <div
      v-for="(symbolHistoryItem, index) in symbolHistory"
      :key="symbolHistoryItem.symbol + index"
    >
      {{ JSON.stringify(symbolHistoryItem) }}
    </div>
  </code>
</template>

<script>
/** core **/
import { reactive, ref } from "vue";

/** components **/

export default {
  name: "App",

  components: {},

  setup() {
    const isOnline = ref(false);
    const ws = reactive({});

    const symbolList = process?.env?.VUE_APP_SYMBOL_LIST?.split(",");
    const symbols = reactive(symbolList);
    const history = reactive(
      symbolList.reduce((acc, item) => {
        acc[item] = [];
        return acc;
      }, {})
    );

    return {
      // Конечно, в реальном приложении мы не должны хранить
      // любые чувствительные данные на фронте (ключи, токены, ...).
      // В бою, мы бы проксировали запросы через собственный сервер
      // или выдавали ключ к API только аутентифицированным клиентам.
      API_KEY: process?.env?.VUE_APP_API_KEY,
      API_URL: process?.env?.VUE_APP_API_URL,
      symbols,
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
          console.log("WebSocket connected");
          this.ws.send(
            `{"userKey":"${this.API_KEY}", "symbol":"${this.symbols.join()}"}`
          );
          this.isOnline = true;
        };

        this.ws.onmessage = msg => {
          if (msg?.data !== "Connected") {
            const messageData = JSON.parse(msg?.data);
            const symbol = messageData?.symbol;

            if (symbol) {
              if (!this.history[symbol]) {
                // Вообще, мы уже инициализировали отдельные свойства под каждый symbol в setup,
                // но на тот случай, если новые symbol будут добавлены в runtime
                this.history[symbol] = [];
              }

              // новые сообщения вставляем в начало массива
              this.history[symbol].unshift(messageData);

              if (this.history[symbol].length > 10) {
                // храним только последние 10 сообщений
                this.history[symbol].pop();
              }
            }
          }
        };

        this.ws.onclose = () => {
          console.warn("WebSocket connection closed");
          // TODO: add reconnect
          this.isOnline = false;
        };

        this.ws.onerror = err => {
          console.error("WebSocket got en error", err);
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
    /**
     * Возвращает актуальный курс для каждого symbol (instrument / currency pair)
     * @returns {*[]}
     */
    currentRateList() {
      return this.symbols.map(symbol => {
        return this.history[symbol]?.length
          ? this.history[symbol][0] // в начале массива хранится наиболее свежая информация
          : { symbol };
      });
    }
  }
};
</script>

<style scoped lang="scss">
body {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-top: 64px;
}

h1,
h2,
h3,
code {
  text-align: center;
}

.connection {
  color: red;
  font-size: 0.6em;

  &.__online {
    color: green;
  }
}

.rates-container {
  max-width: 320px;
  margin: 24px auto;

  .row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);

    &:nth-child(2n + 2) {
      background: lightgrey;
    }
  }

  .col {
    padding: 4px 8px;
  }
}
</style>
