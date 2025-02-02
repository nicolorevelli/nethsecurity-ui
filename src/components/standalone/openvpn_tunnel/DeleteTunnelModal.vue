<script setup lang="ts">
import { NeModal, NeInlineNotification, getAxiosErrorMessage } from '@nethserver/vue-tailwind-lib'
import { ref, toRefs } from 'vue'
import { useI18n } from 'vue-i18n'
import type { ServerTunnel, ClientTunnel } from './TunnelManager.vue'
import { ubusCall } from '@/lib/standalone/ubus'

const { t } = useI18n()

const props = defineProps<{
  visible: boolean
  itemToDelete: ServerTunnel | ClientTunnel | null
}>()

const emit = defineEmits(['close', 'tunnel-deleted'])

const { visible, itemToDelete } = toRefs(props)
const error = ref('')
const isDeleting = ref(false)

async function deleteTunnel() {
  if (itemToDelete.value) {
    try {
      error.value = ''
      isDeleting.value = true
      await ubusCall('ns.ovpntunnel', 'delete-tunnel', { id: itemToDelete.value.id })
      emit('tunnel-deleted')
      emit('close')
    } catch (err: any) {
      error.value = t(getAxiosErrorMessage(err))
    } finally {
      isDeleting.value = false
    }
  }
}

function close() {
  error.value = ''
  emit('close')
}
</script>

<template>
  <NeModal
    :visible="visible"
    kind="warning"
    :title="t('standalone.openvpn_tunnel.delete_tunnel')"
    :primaryLabel="t('common.delete')"
    :primaryButtonDisabled="isDeleting"
    :primaryButtonLoading="isDeleting"
    @primaryClick="deleteTunnel()"
    @close="close()"
  >
    {{
      t('standalone.openvpn_tunnel.delete_tunnel_message', {
        name: itemToDelete?.ns_name ?? ''
      })
    }}
    <NeInlineNotification
      v-if="error"
      kind="error"
      :title="t('error.cannot_delete_tunnel')"
      :description="error"
      class="my-2"
    />
  </NeModal>
</template>
