<template>
  <AppPaneLayout>
    <template #primary>
      <div
        class="sticky top-0 z-10 flex flex-shrink-0 p-4 overflow-x-auto space-x-2 bg-primary hide-scrollbar"
      >
        <div class="inline-flex flex-1 space-x-2">
          <div class="flex flex-1">
            <input
              id="server"
              v-model="server"
              type="url"
              autocomplete="off"
              :class="{ error: !isUrlValid }"
              class="flex flex-1 w-full px-4 py-2 border rounded-l bg-primaryLight border-divider text-secondaryDark"
              :placeholder="$t('sse.url')"
              :disabled="
                connectionState === 'STARTED' || connectionState === 'STARTING'
              "
              @keyup.enter="isUrlValid ? toggleSSEConnection() : null"
            />
            <label
              for="event-type"
              class="px-4 py-2 font-semibold truncate border-t border-b bg-primaryLight border-divider text-secondaryLight"
            >
              {{ $t("sse.event_type") }}
            </label>
            <input
              id="event-type"
              v-model="eventType"
              class="flex flex-1 w-full px-4 py-2 border rounded-r bg-primaryLight border-divider text-secondaryDark"
              spellcheck="false"
              :disabled="
                connectionState === 'STARTED' || connectionState === 'STARTING'
              "
              @keyup.enter="isUrlValid ? toggleSSEConnection() : null"
            />
          </div>
          <ButtonPrimary
            id="start"
            :disabled="!isUrlValid"
            name="start"
            class="w-32"
            :label="
              connectionState === 'STOPPED'
                ? t('action.start')
                : t('action.stop')
            "
            :loading="connectionState === 'STARTING'"
            @click.native="toggleSSEConnection"
          />
        </div>
      </div>
    </template>
    <template #secondary>
      <RealtimeLog
        :title="$t('sse.log')"
        :log="log"
        @delete="clearLogEntries()"
      />
    </template>
  </AppPaneLayout>
</template>

<script setup lang="ts">
import { ref, watch, onUnmounted, onMounted } from "@nuxtjs/composition-api"
import "splitpanes/dist/splitpanes.css"
import debounce from "lodash/debounce"
import {
  SSEEndpoint$,
  setSSEEndpoint,
  SSEEventType$,
  setSSEEventType,
  SSESocket$,
  setSSESocket,
  SSELog$,
  setSSELog,
  addSSELogLine,
} from "~/newstore/SSESession"
import {
  useNuxt,
  useStream,
  useToast,
  useI18n,
  useStreamSubscriber,
  useReadonlyStream,
} from "~/helpers/utils/composables"
import { SSEConnection } from "~/helpers/realtime/SSEConnection"

const t = useI18n()
const nuxt = useNuxt()
const toast = useToast()
const { subscribeToStream } = useStreamSubscriber()

const sse = useStream(SSESocket$, new SSEConnection(), setSSESocket)
const connectionState = useReadonlyStream(sse.value.connectionState$, "STOPPED")
const server = useStream(SSEEndpoint$, "", setSSEEndpoint)
const eventType = useStream(SSEEventType$, "", setSSEEventType)
const log = useStream(SSELog$, [], setSSELog)

const isUrlValid = ref(true)

let worker: Worker

const debouncer = debounce(function () {
  worker.postMessage({ type: "sse", url: server.value })
}, 1000)

watch(server, (url) => {
  if (url) debouncer()
})

const workerResponseHandler = ({
  data,
}: {
  data: { url: string; result: boolean }
}) => {
  if (data.url === server.value) isUrlValid.value = data.result
}

onMounted(() => {
  worker = nuxt.value.$worker.createRejexWorker()
  worker.addEventListener("message", workerResponseHandler)

  subscribeToStream(sse.value.event$, (event) => {
    switch (event?.type) {
      case "STARTING":
        log.value = [
          {
            payload: `${t("state.connecting_to", { name: server.value })}`,
            source: "info",
            color: "var(--accent-color)",
            ts: undefined,
          },
        ]
        break

      case "STARTED":
        log.value = [
          {
            payload: `${t("state.connected_to", { name: server.value })}`,
            source: "info",
            color: "var(--accent-color)",
            ts: Date.now(),
          },
        ]
        toast.success(`${t("state.connected")}`)
        break

      case "MESSAGE_RECEIVED":
        addSSELogLine({
          payload: event.message,
          source: "server",
          ts: event.time,
        })
        break

      case "ERROR":
        addSSELogLine({
          payload: t("error.browser_support_sse").toString(),
          source: "info",
          color: "#ff5555",
          ts: event.time,
        })
        break

      case "STOPPED":
        addSSELogLine({
          payload: t("state.disconnected_from", {
            name: server.value,
          }).toString(),
          source: "info",
          color: "#ff5555",
          ts: event.time,
        })
        toast.error(`${t("state.disconnected")}`)
        break
    }
  })
})

// METHODS

const toggleSSEConnection = () => {
  // If it is connecting:
  if (connectionState.value === "STOPPED") {
    return sse.value.start(server.value, eventType.value)
  }
  // Otherwise, it's disconnecting.
  sse.value.stop()
}

onUnmounted(() => {
  worker.terminate()
})
const clearLogEntries = () => {
  log.value = []
}
</script>
