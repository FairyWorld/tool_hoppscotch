<template>
  <div class="flex flex-col" :class="[{ 'bg-primaryLight': dragging }]">
    <div
      class="flex items-stretch group"
      draggable="true"
      @dragstart="dragStart"
      @dragover.stop
      @dragleave="dragging = false"
      @dragend="dragging = false"
      @contextmenu.prevent="options.tippy.show()"
    >
      <span
        class="flex items-center justify-center w-16 px-2 truncate cursor-pointer"
        :class="getRequestLabelColor(request.method)"
        @click="selectRequest()"
      >
        <component
          :is="IconCheckCircle"
          v-if="isSelected"
          class="svg-icons"
          :class="{ 'text-accent': isSelected }"
        />
        <span v-else class="font-semibold truncate text-tiny">
          {{ request.method }}
        </span>
      </span>
      <span
        class="flex items-center flex-1 min-w-0 py-2 pr-2 cursor-pointer transition group-hover:text-secondaryDark"
        @click="selectRequest()"
      >
        <span class="truncate" :class="{ 'text-accent': isSelected }">
          {{ request.name }}
        </span>
        <span
          v-if="isActive"
          v-tippy="{ theme: 'tooltip' }"
          class="relative h-1.5 w-1.5 flex flex-shrink-0 mx-3"
          :title="`${t('collection.request_in_use')}`"
        >
          <span
            class="absolute inline-flex flex-shrink-0 w-full h-full bg-green-500 rounded-full opacity-75 animate-ping"
          >
          </span>
          <span
            class="relative inline-flex flex-shrink-0 rounded-full h-1.5 w-1.5 bg-green-500"
          ></span>
        </span>
      </span>
      <div class="flex">
        <ButtonSecondary
          v-if="!saveRequest"
          v-tippy="{ theme: 'tooltip' }"
          :icon="IconRotateCCW"
          :title="t('action.restore')"
          class="hidden group-hover:inline-flex"
          @click="selectRequest()"
        />
        <span>
          <tippy
            ref="options"
            interactive
            trigger="click"
            theme="popover"
            arrow
            :on-shown="() => tippyActions.focus()"
          >
            <ButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="t('action.more')"
              :icon="IconMoreVertical"
            />
            <template #content="{ hide }">
              <div
                ref="tippyActions"
                class="flex flex-col focus:outline-none"
                tabindex="0"
                role="menu"
                @keyup.e="edit.$el.click()"
                @keyup.d="duplicate.$el.click()"
                @keyup.delete="deleteAction.$el.click()"
                @keyup.escape="hide()"
              >
                <SmartItem
                  ref="edit"
                  :icon="IconEdit"
                  :label="t('action.edit')"
                  :shortcut="['E']"
                  @click="
                    () => {
                      emit('edit-request', {
                        collectionIndex,
                        folderIndex,
                        folderName,
                        request,
                        requestIndex,
                        folderPath,
                      })
                      hide()
                    }
                  "
                />
                <SmartItem
                  ref="duplicate"
                  :icon="IconCopy"
                  :label="t('action.duplicate')"
                  :shortcut="['D']"
                  @click="
                    () => {
                      emit('duplicate-request', {
                        collectionIndex,
                        folderIndex,
                        folderName,
                        request,
                        requestIndex,
                        folderPath,
                      })
                      hide()
                    }
                  "
                />
                <SmartItem
                  ref="deleteAction"
                  :icon="IconTrash2"
                  :label="t('action.delete')"
                  :shortcut="['⌫']"
                  @click="
                    () => {
                      removeRequest()
                      hide()
                    }
                  "
                />
              </div>
            </template>
          </tippy>
        </span>
      </div>
    </div>
    <HttpReqChangeConfirmModal
      :show="confirmChange"
      @hide-modal="confirmChange = false"
      @save-change="saveRequestChange"
      @discard-change="discardRequestChange"
    />
    <CollectionsSaveRequest
      mode="rest"
      :show="showSaveRequestModal"
      @hide-modal="showSaveRequestModal = false"
    />
  </div>
</template>

<script setup lang="ts">
import IconCheckCircle from "~icons/lucide/check-circle"
import IconMoreVertical from "~icons/lucide/more-vertical"
import IconEdit from "~icons/lucide/edit"
import IconCopy from "~icons/lucide/copy"
import IconTrash2 from "~icons/lucide/trash-2"
import IconRotateCCW from "~icons/lucide/rotate-ccw"
import { ref, computed } from "vue"
import {
  HoppRESTRequest,
  safelyExtractRESTRequest,
  translateToNewRequest,
  isEqualHoppRESTRequest,
} from "@hoppscotch/data"
import * as E from "fp-ts/Either"
import { cloneDeep } from "lodash-es"
import { useI18n } from "@composables/i18n"
import { useToast } from "@composables/toast"
import { useReadonlyStream } from "@composables/stream"
import {
  getDefaultRESTRequest,
  getRESTRequest,
  restSaveContext$,
  setRESTRequest,
  setRESTSaveContext,
  getRESTSaveContext,
} from "~/newstore/RESTSession"
import { editRESTRequest } from "~/newstore/collections"
import { runMutation } from "~/helpers/backend/GQLClient"
import { UpdateRequestDocument } from "~/helpers/backend/graphql"
import { HoppRequestSaveContext } from "~/helpers/types/HoppRequestSaveContext"

const props = defineProps<{
  request: HoppRESTRequest
  collectionIndex: number
  folderIndex: number
  folderName: string
  requestIndex: number
  saveRequest: boolean
  collectionsType: object
  folderPath: string
  picked?: {
    pickedType: string
    collectionIndex: number
    folderPath: string
    folderName: string
    requestIndex: number
  }
}>()

const emit = defineEmits<{
  (
    e: "select",
    data:
      | {
          picked: {
            pickedType: string
            collectionIndex: number
            folderPath: string
            folderName: string
            requestIndex: number
          }
        }
      | undefined
  ): void

  (
    e: "remove-request",
    data: {
      folderPath: string
      requestIndex: number
    }
  ): void

  (
    e: "duplicate-request",
    data: {
      collectionIndex: number
      folderIndex: number
      folderName: string
      request: HoppRESTRequest
      folderPath: string
      requestIndex: number
    }
  ): void

  (
    e: "edit-request",
    data: {
      collectionIndex: number
      folderIndex: number
      folderName: string
      request: HoppRESTRequest
      folderPath: string
      requestIndex: number
    }
  ): void
}>()

const t = useI18n()
const toast = useToast()

const dragging = ref(false)
const requestMethodLabels = {
  get: "text-green-500",
  post: "text-yellow-500",
  put: "text-blue-500",
  delete: "text-red-500",
  default: "text-gray-500",
}
const confirmChange = ref(false)
const showSaveRequestModal = ref(false)

// Template refs
const tippyActions = ref<any | null>(null)
const options = ref<any | null>(null)
const edit = ref<any | null>(null)
const duplicate = ref<any | null>(null)
const deleteAction = ref<any | null>(null)

const active = useReadonlyStream(restSaveContext$, null)

const isSelected = computed(
  () =>
    props.picked &&
    props.picked.pickedType === "my-request" &&
    props.picked.folderPath === props.folderPath &&
    props.picked.requestIndex === props.requestIndex
)

const isActive = computed(
  () =>
    active.value &&
    active.value.originLocation === "user-collection" &&
    active.value.folderPath === props.folderPath &&
    active.value.requestIndex === props.requestIndex
)

const dragStart = ({ dataTransfer }: DragEvent) => {
  if (dataTransfer) {
    dragging.value = !dragging.value
    dataTransfer.setData("folderPath", props.folderPath)
    dataTransfer.setData("requestIndex", props.requestIndex.toString())
  }
}

const removeRequest = () => {
  emit("remove-request", {
    folderPath: props.folderPath,
    requestIndex: props.requestIndex,
  })
}

const getRequestLabelColor = (method: string) =>
  requestMethodLabels[
    method.toLowerCase() as keyof typeof requestMethodLabels
  ] || requestMethodLabels.default

const setRestReq = (request: any) => {
  setRESTRequest(
    cloneDeep(
      safelyExtractRESTRequest(
        translateToNewRequest(request),
        getDefaultRESTRequest()
      )
    ),
    {
      originLocation: "user-collection",
      folderPath: props.folderPath,
      requestIndex: props.requestIndex,
      req: cloneDeep(request),
    }
  )
}

/** Loads request from the save once, checks for unsaved changes, but ignores default values */
const selectRequest = () => {
  // Check if this is a save as request popup, if so we don't need to prompt the confirm change popup.
  if (props.saveRequest) {
    emit("select", {
      picked: {
        pickedType: "my-request",
        collectionIndex: props.collectionIndex,
        folderPath: props.folderPath,
        folderName: props.folderName,
        requestIndex: props.requestIndex,
      },
    })
  } else if (isEqualHoppRESTRequest(props.request, getDefaultRESTRequest())) {
    confirmChange.value = false
    setRestReq(props.request)
  } else if (!active.value) {
    // If the current request is the same as the request to be loaded in, there is no data loss
    const currentReq = getRESTRequest()

    if (isEqualHoppRESTRequest(currentReq, props.request)) {
      setRestReq(props.request)
    } else {
      confirmChange.value = true
    }
  } else {
    const currentReqWithNoChange = active.value.req
    const currentFullReq = getRESTRequest()

    // Check if whether user clicked the same request or not
    if (!isActive.value && currentReqWithNoChange !== undefined) {
      // Check if there is any changes done on the current request
      if (isEqualHoppRESTRequest(currentReqWithNoChange, currentFullReq)) {
        setRestReq(props.request)
      } else {
        confirmChange.value = true
      }
    } else {
      setRESTSaveContext(null)
    }
  }
}

/** Save current request to the collection */
const saveRequestChange = () => {
  const saveCtx = getRESTSaveContext()
  saveCurrentRequest(saveCtx)
  confirmChange.value = false
}

/** Discard changes and change the current request and context */
const discardRequestChange = () => {
  setRestReq(props.request)
  if (!isActive.value) {
    setRESTSaveContext({
      originLocation: "user-collection",
      folderPath: props.folderPath,
      requestIndex: props.requestIndex,
      req: cloneDeep(props.request),
    })
  }

  confirmChange.value = false
}

const saveCurrentRequest = (saveCtx: HoppRequestSaveContext | null) => {
  if (!saveCtx) {
    showSaveRequestModal.value = true
    return
  }
  if (saveCtx.originLocation === "user-collection") {
    try {
      editRESTRequest(
        saveCtx.folderPath,
        saveCtx.requestIndex,
        getRESTRequest()
      )
      setRestReq(props.request)
      toast.success(`${t("request.saved")}`)
    } catch (e) {
      setRESTSaveContext(null)
      saveCurrentRequest(saveCtx)
    }
  } else if (saveCtx.originLocation === "team-collection") {
    const req = getRESTRequest()
    try {
      runMutation(UpdateRequestDocument, {
        requestID: saveCtx.requestID,
        data: {
          title: req.name,
          request: JSON.stringify(req),
        },
      })().then((result) => {
        if (E.isLeft(result)) {
          toast.error(`${t("profile.no_permission")}`)
        } else {
          toast.success(`${t("request.saved")}`)
        }
      })
      setRestReq(props.request)
    } catch (error) {
      showSaveRequestModal.value = true
      toast.error(`${t("error.something_went_wrong")}`)
      console.error(error)
    }
  }
}
</script>
