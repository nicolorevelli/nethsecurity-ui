<script setup lang="ts">
import { ref, toRefs } from 'vue'
import {
  MessageBag,
  validateHostname,
  validateIpAddress,
  validateMacAddress,
  validateRequired,
  type validationOutput
} from '@/lib/validation'
import { onMounted } from 'vue'
import { watchEffect } from 'vue'
import {
  NeSideDrawer,
  NeInlineNotification,
  NeTextInput,
  NeButton,
  getAxiosErrorMessage
} from '@nethserver/vue-tailwind-lib'
import { useI18n } from 'vue-i18n'
import { ubusCall } from '@/lib/standalone/ubus'
import type { StaticLease } from './StaticLeases.vue'

const props = defineProps<{
  isShown: boolean
  itemToEdit: StaticLease | null
}>()
const { isShown } = toRefs(props)

const { t } = useI18n()

const emit = defineEmits(['close', 'add-edit-lease'])

const isSavingChanges = ref(false)
const error = ref({
  notificationTitle: '',
  notificationDescription: ''
})
const validationErrorBag = ref(new MessageBag())

// Form fields
const id = ref('')
const hostname = ref('')
const ipAddress = ref('')
const macAddress = ref('')
const reservationName = ref('')

function resetForm() {
  id.value = props.itemToEdit?.lease ?? ''
  hostname.value = props.itemToEdit?.hostname ?? ''
  ipAddress.value = props.itemToEdit?.ipaddr ?? ''
  macAddress.value = props.itemToEdit?.macaddr ?? ''
  reservationName.value = props.itemToEdit?.description ?? ''
}

function runValidators(validators: validationOutput[], label: string): boolean {
  for (let validator of validators) {
    if (!validator.valid) {
      validationErrorBag.value.set(label, [t(validator.errMessage as string)])
    }
  }

  return validators.every((validator) => validator.valid)
}

function validate() {
  validationErrorBag.value.clear()

  const validators: [validationOutput[], string][] = [
    [[validateRequired(hostname.value), validateHostname(hostname.value)], 'hostname'],
    [[validateRequired(ipAddress.value), validateIpAddress(ipAddress.value)], 'ipAddress'],
    [[validateRequired(macAddress.value), validateMacAddress(macAddress.value)], 'macAddress']
  ]

  return validators
    .map(([validator, label]) => runValidators(validator, label))
    .every((result) => result)
}

async function createOrEditStaticLease() {
  error.value.notificationTitle = ''
  error.value.notificationDescription = ''
  const isEditing = id.value != ''

  try {
    isSavingChanges.value = true
    const requestType = isEditing ? 'edit-static-lease' : 'add-static-lease'

    if (validate()) {
      let payload: {
        lease?: string
        hostname: string
        ipaddr: string
        description: string
        macaddr: string
      } = {
        hostname: hostname.value,
        ipaddr: ipAddress.value,
        description: reservationName.value,
        macaddr: macAddress.value
      }

      if (isEditing) payload.lease = id.value

      await ubusCall('ns.dhcp', requestType, payload)
      emit('add-edit-lease')
      close()
    }
  } catch (err: any) {
    error.value.notificationTitle = isEditing
      ? t('error.cannot_edit_reservation')
      : t('error.cannot_create_reservation')

    error.value.notificationDescription =
      err.response.data.message == 'lease_not_found'
        ? t('standalone.dns_dhcp.lease_not_found')
        : t(getAxiosErrorMessage(err))
  } finally {
    isSavingChanges.value = false
  }
}

function close() {
  validationErrorBag.value.clear()
  error.value.notificationTitle = ''
  error.value.notificationDescription = ''
  resetForm()
  emit('close')
}

watchEffect(() => {
  resetForm()
})

onMounted(() => {
  resetForm()
})
</script>

<template>
  <NeSideDrawer
    :is-shown="isShown"
    @close="close()"
    :closeAriaLabel="t('standalone.shell.close_side_drawer')"
    :title="
      id ? t('standalone.dns_dhcp.edit_reservation') : t('standalone.dns_dhcp.add_reservation')
    "
  >
    <NeInlineNotification
      v-if="error.notificationTitle"
      :title="error.notificationTitle"
      :description="error.notificationDescription"
      class="mb-6"
      kind="error"
    />
    <div class="flex flex-col gap-y-6">
      <NeTextInput
        v-model="hostname"
        :label="t('standalone.dns_dhcp.hostname')"
        :invalid-message="validationErrorBag.getFirstFor('hostname')"
      />
      <NeTextInput
        v-model="ipAddress"
        :label="t('standalone.dns_dhcp.ip_address')"
        :invalid-message="validationErrorBag.getFirstFor('ipAddress')"
      />
      <NeTextInput
        v-model="macAddress"
        :label="t('standalone.dns_dhcp.mac_address')"
        :invalid-message="validationErrorBag.getFirstFor('macAddress')"
      />
      <NeTextInput
        v-model="reservationName"
        :label="t('standalone.dns_dhcp.reservation_name')"
        :optional="true"
        :optional-label="t('common.optional')"
      />
      <hr />
      <div class="flex justify-end">
        <NeButton kind="tertiary" class="mr-4" @click="close()">{{ t('common.cancel') }}</NeButton>
        <NeButton
          kind="primary"
          @click="createOrEditStaticLease()"
          :disabled="isSavingChanges"
          :loading="isSavingChanges"
          >{{ t('common.save') }}</NeButton
        >
      </div>
    </div></NeSideDrawer
  >
</template>
