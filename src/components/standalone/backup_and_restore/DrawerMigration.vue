<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { ubusCall } from '@/lib/standalone/ubus'
import {
  NeModal,
  NeTitle,
  NeButton,
  NeFileInput,
  NeSideDrawer,
  NeProgressBar,
  NeInlineNotification,
  getAxiosErrorMessage,
  NeFormItemLabel,
  NeSkeleton,
  NeCombobox,
  NeTextInput
} from '@nethserver/vue-tailwind-lib'
import { validateRequired } from '@/lib/validation'
const { t } = useI18n()

defineProps({
  showMigrationDrawer: {
    type: Boolean,
    required: true
  }
})

const MIGRATION_WAIT_TIME = 25000
const emit = defineEmits(['success', 'close'])
const formMigration = ref({
  file: undefined,
  devices: []
})

const loading = ref(false)
const loadingFile = ref(false)
const loadingMigration = ref(false)
const isMigrating = ref(false)
const migrationProgress = ref(0)
const listDevices = ref([])
const listDevicesMigration = ref([
  {
    id: '',
    label: '',
    selected: ''
  }
])
const migrationIntervalRef = ref<number | undefined>()
const fileRef = ref()

let errorMigration = ref({
  file: '',
  devices: ['']
})
let errorMigrationFile = ref({
  notificationTitle: '',
  notificationDescription: '',
  notificationDetails: ''
})
let errorMigrationBackup = ref({
  notificationTitle: '',
  notificationDescription: '',
  notificationDetails: ''
})
let errorLoadDevices = ref({
  notificationTitle: '',
  notificationDescription: '',
  notificationDetails: ''
})

onMounted(() => {
  getListDevices()
})

watch(
  () => formMigration.value.file,
  async () => {
    errorMigrationFile.value = {
      notificationTitle: '',
      notificationDescription: '',
      notificationDetails: ''
    }
    if (formMigration.value.file) {
      loadingFile.value = true
      await new Promise((resolve: any) => {
        let reader = new FileReader()
        reader.onload = async function (event) {
          if (event?.target?.result) {
            try {
              let payload = {
                archive: String(event.target.result).split(',')[1]
              }

              let res = await ubusCall('ns.migration', 'upload', payload)
              if (res?.data?.devices?.length) {
                listDevicesMigration.value = res.data.devices.map((item: any) => ({
                  id: item.hwaddr,
                  selected: undefined,
                  label:
                    item.name +
                    (item.ipaddr ? ' - ' + item.ipaddr : '') +
                    (item.role ? ' - ' + item.role : '')
                }))
              }
            } catch (exception: any) {
              errorMigrationFile.value.notificationTitle = t('error.upload_file_migration')
              errorMigrationFile.value.notificationDescription = t(getAxiosErrorMessage(exception))
              errorMigrationFile.value.notificationDetails = exception.toString()
            } finally {
              loadingFile.value = false
            }
          }
          resolve()
        }
        if (formMigration.value.file) {
          reader.readAsDataURL(formMigration.value.file)
        }
      })
    }
  }
)

async function getListDevices() {
  loading.value = true
  try {
    let res = await ubusCall('ns.migration', 'list-devices', {})
    if (res?.data?.devices?.length) {
      listDevices.value = res.data.devices.map((item: any) => ({
        id: item.hwaddr,
        label:
          item.name +
          (item.ipaddr ? ' - ' + item.ipaddr : '') +
          (item.role ? ' - ' + item.role : ''),
        role: item.role,
        hwaddr: item.hwaddr
      }))
    }
  } catch (exception: any) {
    errorLoadDevices.value.notificationTitle = t('error.cannot_retrieve_devices')
    errorLoadDevices.value.notificationDescription = t(getAxiosErrorMessage(exception))
    errorLoadDevices.value.notificationDetails = exception.toString()
  } finally {
    loading.value = false
  }
}

function validateMigration(): boolean {
  let isValidationOk = true

  let { valid, errMessage } = validateRequired(
    formMigration?.value?.file ? formMigration.value.file : ''
  )
  if (!valid) {
    errorMigration.value.file = t(errMessage as string)
    isValidationOk = false
  }

  if (listDevicesMigration.value.length) {
    for (let [index, item] of listDevicesMigration.value.entries()) {
      if (!item.selected) {
        let { valid, errMessage } = validateRequired(item.selected)
        if (!valid) {
          errorMigration.value.devices[index] = t(errMessage as string)
          isValidationOk = false
        }
      }
    }
  }

  return isValidationOk
}

function clearErrors() {
  migrationIntervalRef.value = undefined
  migrationProgress.value = 0
  errorMigration.value = {
    file: '',
    devices: ['']
  }
}

async function startMigration() {
  clearErrors()
  if (validateMigration()) {
    loadingMigration.value = true
    try {
      let devices = []
      if (listDevicesMigration.value.length) {
        for (let item of listDevicesMigration.value) {
          devices.push({
            old: item.id,
            new: item.selected
          })
        }
      }

      let payload = {
        mappings: devices
      }

      emit('close')
      isMigrating.value = true
      setMigrationTimer()

      let res = await ubusCall('ns.migration', 'migrate', payload)
      if (res?.data?.result && res?.data?.result === 'success') {
        isMigrating.value = false
        emit('success')
      }
    } catch (exception: any) {
      errorMigrationBackup.value.notificationTitle = t('error.cannot_migrate')
      errorMigrationBackup.value.notificationDescription = t(getAxiosErrorMessage(exception))
      errorMigrationBackup.value.notificationDetails = exception.toString()
    } finally {
      loadingMigration.value = false
    }
  }
}

function setMigrationTimer() {
  migrationIntervalRef.value = setInterval(() => {
    migrationProgress.value += 0.5
  }, MIGRATION_WAIT_TIME / 200)
}
</script>

<template>
  <div>
    <NeSideDrawer :is-shown="showMigrationDrawer" title="" @close="$emit('close')">
      <NeSkeleton v-if="loading" :lines="5" />
      <NeInlineNotification
        v-if="!loading && errorLoadDevices.notificationTitle"
        class="my-4"
        kind="error"
        :title="errorLoadDevices.notificationTitle"
        :description="errorLoadDevices.notificationDescription"
      >
        <template v-if="errorLoadDevices.notificationDetails" #details>
          {{ errorLoadDevices.notificationDetails }}
        </template>
      </NeInlineNotification>
      <div v-if="!loading && !errorLoadDevices.notificationTitle" class="space-y-5">
        <NeTitle>{{ t('standalone.backup_and_restore.migration.drawer_title') }}</NeTitle>
        <hr />
        <NeSkeleton v-if="loadingFile" :lines="5" />
        <NeFileInput
          v-if="!loadingFile"
          :label="t('standalone.backup_and_restore.migration.input_upload_file')"
          :dropzoneLabel="
            t('standalone.backup_and_restore.migration.input_upload_file_description')
          "
          :invalid-message="errorMigration.file"
          v-model="formMigration.file"
          ref="fileRef"
        />
        <NeInlineNotification
          v-if="errorMigrationFile.notificationTitle"
          class="my-4"
          kind="error"
          :title="errorMigrationFile.notificationTitle"
          :description="errorMigrationFile.notificationDescription"
        >
          <template v-if="errorMigrationFile.notificationDetails" #details>
            {{ errorMigrationFile.notificationDetails }}
          </template>
        </NeInlineNotification>
        <template v-if="formMigration.file && !loadingFile && listDevicesMigration.length">
          <div>
            <NeFormItemLabel>
              {{ t('standalone.backup_and_restore.migration.remap_interfaces') }}
            </NeFormItemLabel>
          </div>
          <div v-for="(deviceMigration, index) in listDevicesMigration" :key="index">
            <div class="flex items-center">
              <div class="flex-grow">
                <NeTextInput v-model="deviceMigration.label" disabled />
              </div>
              <div class="px-3 text-center text-sm">
                {{ t('standalone.backup_and_restore.migration.to') }}
              </div>
              <div class="flex-grow">
                <NeCombobox
                  v-model="deviceMigration.selected"
                  class="grow"
                  :options="listDevices"
                  :invalid-message="errorMigration.devices[index]"
                  :ref="'deviceMigration' + index"
                />
              </div>
            </div>
          </div>
          <NeInlineNotification
            kind="info"
            :title="t('standalone.backup_and_restore.migration.remap_interfaces_description')"
          />
        </template>
        <hr />
        <NeInlineNotification
          v-if="errorMigrationBackup.notificationTitle"
          class="my-4"
          kind="error"
          :title="errorMigrationBackup.notificationTitle"
          :description="errorMigrationBackup.notificationDescription"
        >
          <template v-if="errorMigrationBackup.notificationDetails" #details>
            {{ errorMigrationBackup.notificationDetails }}
          </template>
        </NeInlineNotification>
        <div class="flex justify-end gap-4">
          <NeButton :disabled="loadingMigration" :kind="'tertiary'" @click="$emit('close')">
            {{ t('common.cancel') }}
          </NeButton>
          <NeButton
            :disabled="loadingMigration"
            :kind="'primary'"
            :loading="loadingMigration"
            @click="startMigration()"
          >
            {{ t('standalone.backup_and_restore.migration.migrate') }}
          </NeButton>
        </div>
      </div>
    </NeSideDrawer>
    <NeModal
      :primary-label="t('common.cancel')"
      :primary-button-disabled="true"
      :title="t('standalone.backup_and_restore.migration.migrate')"
      :visible="isMigrating"
      cancel-label=""
      kind="warning"
      primary-button-kind="danger"
      @close="isMigrating = false"
    >
      {{ t('standalone.backup_and_restore.migration.migration_in_progress') }}
      <NeProgressBar class="my-4" :progress="migrationProgress" />
    </NeModal>
  </div>
</template>
