<template>
  <tr
    :data-pivot-id="resource['id'].pivotValue"
    :dusk="resource['id'].value + '-row'"
    class="group"
    :class="{
      'divide-x divide-gray-100 dark:divide-gray-700': shouldShowColumnBorders,
    }"
    @click.stop.prevent="navigateToDetail"
  >
    <!-- Resource Selection Checkbox -->
    <td
      v-if="shouldShowCheckboxes"
      :class="{
        'py-2': !shouldShowTight,
        'cursor-pointer': resource.authorizedToView,
      }"
      class="td-fit pl-5 pr-5 dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
      @click.stop
    >
      <Checkbox
        v-if="shouldShowCheckboxes"
        :aria-label="__('Select Resource :title', { title: resource.title })"
        :checked="checked"
        :data-testid="`${testId}-checkbox`"
        :dusk="`${resource['id'].value}-checkbox`"
        @input="toggleSelection"
      />
    </td>

    <td
      :class="{
        'py-2': !shouldShowTight,
        'cursor-pointer': resource.authorizedToView,
      }"
      class="px-2 td-fit text-right align-middle dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
    >
      <div class="flex items-center justify-end space-x-0 text-gray-400">
        <InlineActionDropdown
          v-if="availableActions.length > 0"
          :actions="availableActions"
          :endpoint="actionsEndpoint"
          :resource="resource"
          :resource-name="resourceName"
          :via-many-to-many="viaManyToMany"
          :via-resource="viaResource"
          :via-resource-id="viaResourceId"
          :via-relationship="viaRelationship"
          @actionExecuted="$emit('actionExecuted')"
          @show-preview="navigateToPreviewView"
        />

        <!-- View Resource Link -->
        <Link
          v-if="authorizedToViewAnyResources"
          :as="!resource.authorizedToView ? 'button' : 'a'"
          :disabled="!resource.authorizedToView"
          v-tooltip.click="__('View')"
          :aria-label="__('View')"
          :dusk="`${resource['id'].value}-view-button`"
          :href="viewURL"
          class="toolbar-button hover:text-primary-500 px-2 disabled:opacity-50 disabled:pointer-events-none"
          @click.stop
        >
          <Icon type="eye" />
        </Link>

        <!-- Edit Pivot Button -->
        <Link
          v-if="authorizedToUpdateAnyResources && viaManyToMany"
          :as="!resource.authorizedToUpdate ? 'button' : 'a'"
          :disabled="!resource.authorizedToUpdate"
          v-tooltip.click="__('Edit Attached')"
          :aria-label="__('Edit Attached')"
          :dusk="`${resource['id'].value}-edit-attached-button`"
          :href="updateURL"
          class="toolbar-button hover:text-primary-500 px-2 disabled:opacity-50 disabled:pointer-events-none"
          @click.stop
        >
          <Icon type="pencil-alt" />
        </Link>

        <!-- Edit Resource Link -->
        <Link
          v-else-if="authorizedToUpdateAnyResources"
          :as="!resource.authorizedToUpdate ? 'button' : 'a'"
          :disabled="!resource.authorizedToUpdate"
          v-tooltip.click="__('Edit')"
          :aria-label="__('Edit')"
          :dusk="`${resource['id'].value}-edit-button`"
          :href="updateURL"
          class="toolbar-button hover:text-primary-500 px-2 disabled:opacity-50 disabled:pointer-events-none"
          @click.stop
        >
          <Icon type="pencil-alt" />
        </Link>

        <!-- Delete Resource Link -->
        <button
          v-if="
            authorizedToDeleteAnyResources &&
            (!resource.softDeleted || viaManyToMany)
          "
          v-tooltip.click="__(viaManyToMany ? 'Detach' : 'Delete')"
          :aria-label="__(viaManyToMany ? 'Detach' : 'Delete')"
          :data-testid="`${testId}-delete-button`"
          :disabled="!resource.authorizedToDelete"
          :dusk="`${resource['id'].value}-delete-button`"
          class="toolbar-button hover:text-primary-500 px-2 disabled:opacity-50 disabled:pointer-events-none"
          @click.stop="openDeleteModal"
        >
          <Icon type="trash" />
        </button>

        <!-- Restore Resource Link -->
        <button
          v-if="
            authorizedToRestoreAnyResources &&
            resource.softDeleted &&
            !viaManyToMany
          "
          v-tooltip.click="__('Restore')"
          :aria-label="__('Restore')"
          :disabled="!resource.authorizedToRestore"
          :dusk="`${resource['id'].value}-restore-button`"
          class="toolbar-button hover:text-primary-500 px-2 disabled:opacity-50 disabled:pointer-events-none"
          @click.stop="openRestoreModal"
        >
          <Icon type="refresh" />
        </button>

        <DeleteResourceModal
          :mode="viaManyToMany ? 'detach' : 'delete'"
          :show="deleteModalOpen"
          @close="closeDeleteModal"
          @confirm="confirmDelete"
        />

        <RestoreResourceModal
          :show="restoreModalOpen"
          @close="closeRestoreModal"
          @confirm="confirmRestore"
        >
          <ModalHeader v-text="__('Restore Resource')" />
          <ModalContent>
            <p class="leading-normal">
              {{ __('Are you sure you want to restore this resource?') }}
            </p>
          </ModalContent>
        </RestoreResourceModal>
      </div>
    </td>

    <!-- Fields -->
    <td
      v-for="(field, index) in resource.fields"
      :key="field.uniqueKey"
      :class="{
        'px-6': index == 0 && !shouldShowCheckboxes,
        'px-2': index != 0 || shouldShowCheckboxes,
        'py-2': !shouldShowTight,
        'whitespace-nowrap': !field.wrapping,
        'cursor-pointer': clickableRow,
      }"
      class="dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
    >
      <component
        :is="'index-' + field.component"
        :class="`text-${field.textAlign}`"
        :field="field"
        :resource="resource"
        :resource-name="resourceName"
        :via-resource="viaResource"
        :via-resource-id="viaResourceId"
      />
    </td>

  </tr>

  <PreviewResourceModal
    v-if="previewModalOpen"
    :resource-id="resource.id.value"
    :resource-name="resourceName"
    :show="previewModalOpen"
    @close="closePreviewModal"
    @confirm="closePreviewModal"
  />
</template>

<script>
import filter from 'lodash/filter'
import { Inertia } from '@inertiajs/inertia'

export default {
  emits: ['actionExecuted'],

  inject: [
    'authorizedToViewAnyResources',
    'authorizedToUpdateAnyResources',
    'authorizedToDeleteAnyResources',
    'authorizedToRestoreAnyResources',
  ],

  props: [
    'testId',
    'deleteResource',
    'restoreResource',
    'resource',
    'resourcesSelected',
    'resourceName',
    'relationshipType',
    'viaRelationship',
    'viaResource',
    'viaResourceId',
    'viaManyToMany',
    'checked',
    'actionsAreAvailable',
    'actionsEndpoint',
    'shouldShowCheckboxes',
    'shouldShowColumnBorders',
    'tableStyle',
    'updateSelectionStatus',
    'queryString',
    'clickAction',
  ],

  data: () => ({
    commandPressed: false,
    deleteModalOpen: false,
    restoreModalOpen: false,
    previewModalOpen: false,
  }),

  mounted() {
    window.addEventListener('keydown', this.handleKeydown)
    window.addEventListener('keyup', this.handleKeyup)
  },

  beforeUnmount() {
    window.removeEventListener('keydown', this.handleKeydown)
    window.removeEventListener('keyup', this.handleKeyup)
  },

  methods: {
    /**
     * Select the resource in the parent component
     */
    toggleSelection() {
      this.updateSelectionStatus(this.resource)
    },

    handleKeydown(e) {
      if (e.key === 'Meta' || e.key === 'Control') {
        this.commandPressed = true
      }
    },

    handleKeyup(e) {
      if (e.key === 'Meta' || e.key === 'Control') {
        this.commandPressed = false
      }
    },

    navigateToDetail(e) {
      if (this.clickAction === 'edit') {
        return this.navigateToEditView(e)
      } else if (this.clickAction === 'select') {
        return this.toggleSelection()
      } else if (this.clickAction === 'ignore') {
        return
      } else if (this.clickAction === 'detail') {
        return this.navigateToDetailView(e)
      } else if (this.clickAction === 'preview') {
        return this.navigateToPreviewView(e)
      } else {
        return this.navigateToDetailView(e)
      }
    },

    navigateToDetailView(e) {
      if (!this.resource.authorizedToView) {
        return
      }
      this.commandPressed
        ? window.open(this.viewURL, '_blank')
        : Inertia.visit(this.viewURL)
    },

    navigateToEditView(e) {
      if (!this.resource.authorizedToUpdate) {
        return
      }
      this.commandPressed
        ? window.open(this.updateURL, '_blank')
        : Inertia.visit(this.updateURL)
    },

    navigateToPreviewView(e) {
      if (!this.resource.authorizedToView) {
        return
      }
      this.openPreviewModal()
    },

    openPreviewModal() {
      this.previewModalOpen = true
    },

    closePreviewModal() {
      this.previewModalOpen = false
    },

    openDeleteModal() {
      this.deleteModalOpen = true
    },

    confirmDelete() {
      this.deleteResource(this.resource)
      this.closeDeleteModal()
    },

    closeDeleteModal() {
      this.deleteModalOpen = false
    },

    openRestoreModal() {
      this.restoreModalOpen = true
    },

    confirmRestore() {
      this.restoreResource(this.resource)
      this.closeRestoreModal()
    },

    closeRestoreModal() {
      this.restoreModalOpen = false
    },
  },

  computed: {
    updateURL() {
      if (this.viaManyToMany) {
        return this.$url(
          `/resources/${this.viaResource}/${this.viaResourceId}/edit-attached/${this.resourceName}/${this.resource.id.value}`,
          {
            viaRelationship: this.viaRelationship,
            viaPivotId: this.resource.id.pivotValue,
          }
        )
      }

      return this.$url(
        `/resources/${this.resourceName}/${this.resource.id.value}/edit`,
        {
          viaResource: this.viaResource,
          viaResourceId: this.viaResourceId,
          viaRelationship: this.viaRelationship,
        }
      )
    },

    viewURL() {
      return this.$url(
        `/resources/${this.resourceName}/${this.resource.id.value}`
      )
    },

    availableActions() {
      return filter(this.resource.actions, a => a.showOnTableRow)
    },

    shouldShowTight() {
      return this.tableStyle === 'tight'
    },

    clickableRow() {
      if (this.clickAction === 'edit') {
        return this.resource.authorizedToUpdate
      } else if (this.clickAction === 'select') {
        return this.shouldShowCheckboxes
      } else if (this.clickAction === 'ignore') {
        return false
      } else if (this.clickAction === 'detail') {
        return this.resource.authorizedToView
      } else if (this.clickAction === 'preview') {
        return this.resource.authorizedToView
      } else {
        return this.resource.authorizedToView
      }
    },
  },
}
</script>
