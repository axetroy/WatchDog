<template>
  <div class="container">
    <img alt="Logo" src="./assets/logo.svg" style="margin-bottom: 20px" />

    <div
      style="padding: 10px; color: #fff; border-radius: 4px"
      :class="{
        'bg-success': services.every((v) => !v.error),
        'bg-warning': services.some((v) => v.error),
        'bg-error': services.every((v) => v.error),
      }"
    >
      <template v-if="services.every((v) => !v.error)">ALL PASS</template>
      <template v-else-if="services.every((v) => v.error)">ALL ERROR</template>
      <template v-else-if="services.some((v) => v.error)">SOME ERROR</template>
    </div>
    <table style="width: 100%; margin-top: 10px">
      <thead>
        <tr>
          <th>STATUS</th>
          <th>NAME</th>
          <th>DURATION</th>
          <th>TREND</th>
          <th>LATEST CHECK</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="v in services" :key="v.name">
          <td style="position: relative">
            <img
              v-if="v.error"
              style="widht: 30px; height: 30px; vertical-align: middle"
              src="./assets/error.svg"
              :title="v.error"
            />
            <img
              v-else
              style="widht: 30px; height: 30px; vertical-align: middle"
              src="./assets/check.svg"
              title="success"
            />
          </td>
          <td>{{ v.name }}</td>
          <td>{{ formatDuration(v.duration) }}ms</td>
          <td style="width: 300px">
            <speed-line :data-source="hisgory.get(v.name)"></speed-line>
          </td>
          <td>{{ formatDate(v.updated_at) }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { format } from "date-fns";
import SpeedLine from "./components/SpeedLine.vue";

interface Message<T = unknown> {
  event: "init" | "update";
  payload: T;
}

export interface Service {
  name: string; // 服务名称
  error?: string; // 错误信息
  updated_at: string; // 更新日期
  duration: number; // 持续时间
}

const MAX_HISTORY_SIZE = 100; // 最长的消息列表

export default defineComponent({
  components: {
    SpeedLine,
  },
  data: () => {
    const state: {
      services: Service[];
      hisgory: Map<string, Service[]>;
      ws?: WebSocket | null;
    } = {
      services: [],
      hisgory: new Map(),
      ws: null,
    };

    return state;
  },
  methods: {
    formatDate(val: string) {
      return format(new Date(val), "yyyy-MM-dd HH:mm:ss");
    },
    formatDuration(duration: number) {
      return (duration / 1000 / 1000).toFixed(2);
    },
    updateService(s: Service) {
      const service = this.services.find((v) => v.name === s.name);

      if (service) {
        for (const attr in s) {
          // @ts-expect-error ignore
          service[attr] = s[attr];
        }
      } else {
        this.services.push(s);
      }
    },
    connect() {
      if (this.ws) {
        this.ws?.close();
        this.ws = null;
      }

      const host =
        // @ts-expect-error
        process.env.NODE_ENV === "production"
          ? location.host
          : "localhost:9999";

      const ws = new WebSocket(`ws://${host}/api/ws`);

      this.ws = ws;

      ws.onopen = () => {
        console.log("Websocket 已连接");
      };

      ws.onclose = (event) => {
        console.log(
          "Socket is closed. Reconnect will be attempted in 1 second.",
          event.reason
        );
        setTimeout(() => {
          this.connect();
        }, 1000);
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event.data) as Message;

        switch (data.event) {
          case "init":
            {
              const payload = (data as Message<{ [key: string]: Service[] }>)
                .payload;
              const services = Object.keys(payload);
              for (const n of services) {
                this.hisgory.set(n, payload[n]);
                const status = payload[n];
                if (status && status.length) {
                  this.updateService(status[status.length - 1]);
                }
              }
            }
            break;
          case "update":
            {
              const payload = (data as Message<Service>).payload;
              this.updateService(payload);
              const target = this.hisgory.get(payload.name) || [];

              target.push(payload);

              const newHistory =
                target.length < MAX_HISTORY_SIZE
                  ? target
                  : target.slice(target.length - MAX_HISTORY_SIZE, target.length);

              this.hisgory.set(payload.name, newHistory);
            }
            break;
        }
      };
    },
  },
  mounted() {
    this.connect();
  },
  unmounted() {},
});
</script>

<style>
.container {
  width: 960px;
  margin: 0 auto;
}

table {
  border-collapse: collapse;
}

tbody tr {
  border-bottom: 1px solid #e2e2e2;
}

tbody tr td {
  padding: 10px 0;
}

.bg-success {
  background-color: #13ce66;
}

.bg-warning {
  background-color: #ebdd65;
}

.bg-error {
  background-color: #f44336;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
