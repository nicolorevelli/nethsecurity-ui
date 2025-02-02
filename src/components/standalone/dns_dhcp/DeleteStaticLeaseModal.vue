<script setup lang="ts">
import { ubusCall } from '@/lib/standalone/ubus'
import { NeModal, NeInlineNotification, getAxiosErrorMessage } from '@nethserver/vue-tailwind-lib'
import { ref, toRefs } from 'vue'
import { useI18n } from 'vue-i18n'
import type { StaticLease } from './StaticLeases.vue'

const { t } = useI18n()

const props = defineProps<{
  visible: boolean
  itemToDelete: StaticLease | null
}>()

const emit = defineEmits(['close', 'lease-deleted'])

const { visible, itemToDelete } = toRefs(props)
const error = ref('')
const isDeleting = ref(false)

async function deleteStaticLease() {
  if (itemToDelete.value) {
    try {
      error.value = ''
      isDeleting.value = true
      await ubusCall('ns.dhcp', 'delete-static-lease', {
        lease: itemToDelete.value.lease
      })
      emit('lease-deleted')
      emit('close')
    } catch (err: any) {
      error.value =
        err.response.data.message == 'lease_not_found'
          ? t('standalone.dns_dhcp.lease_not_found')
          : t(getAxiosErrorMessage(err))
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
    :title="t('standalone.dns_dhcp.delete_reservation')"
    :primaryLabel="t('common.delete')"
    :primaryButtonDisabled="isDeleting"
    :primaryButtonLoading="isDeleting"
    @primaryClick="deleteStaticLease()"
    @close="close()"
  >
    {{
      t('standalone.dns_dhcp.delete_reservation_message', {
        hostname: itemToDelete?.hostname ?? ''
      })
    }}
    <NeInlineNotification
      v-if="error"
      kind="error"
      :title="t('error.cannot_delete_reservation')"
      :description="error"
    />
  </NeModal>
</template>
