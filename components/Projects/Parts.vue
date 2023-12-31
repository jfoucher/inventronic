<template>
  <UCard>
    <template #header>
      <div class="flex flex-row justify-between">
        <h2 class="text-xl font-bold u-text-white">{{ project.project_parts.length }} project parts</h2>
        <UDropdown :items="items" :popper="{ placement: 'bottom-start' }">
          <UButton color="white" label="Add parts" trailing-icon="i-heroicons-chevron-down-20-solid" />
          <ProjectsUploadModal @change="kicadCSV($event)" :open="uploadModal" @close="uploadModal = false" />
          <ProjectsProjectPartModal :open="newppModal" :projectPart="newppPart" @close="newppModal = false"
            @refresh="refresh" />
        </UDropdown>
      </div>
    </template>
    <div>
      <div :class="selected.length ? '' : 'h-8'">
        <UButtonGroup size="sm" orientation="horizontal" v-if="selected.length">

          <UButton icon="i-heroicons-outline-qr-code" label="Print labels" color="white" @click="printTags" />
          <UButton :loading="deletingAll" icon="i-heroicons-outline-trash" color="red" label="Delete"
            @click="deleteParts" />

        </UButtonGroup>
      </div>
      <!-- TODO display as table only for wide screens -->
      <UTable v-model="selected" :rows="rows" :columns="columns" :ui="{ td: { base: '' } }">
        <template #id-data="{ row }">
          <div class="flex flex-row">
            <div v-if="row.id">
              <UTooltip text="Print project part label">
                <UButton class="mr-2" label="" icon="i-heroicons-outline-qr-code" @click="printTag(row)" />
              </UTooltip>
            </div>
            <div v-if="row.id">
              <UTooltip text="Edit project part">
                <UButton class="mr-2" label="" icon="i-heroicons-outline-pencil" @click="editProjectPart(row)" />
              </UTooltip>
            </div>
            <div v-if="row.id">
              <UTooltip text="Remove part from this project">
                <UButton class="mr-2" :loading="deleting === row.id" label="" color="red"
                  :icon="deleting === row.id ? '' : 'i-heroicons-outline-trash'" @click="removeProjectPart(row)" />
              </UTooltip>
            </div>

            <div v-else-if="row.parts.id">
              <UTooltip text="Add to this project">
                <UButton class="mr-2" label=""
                  :icon="adding[row.parts.part + row.parts.value] ? '' : 'i-heroicons-outline-plus'"
                  :loading="adding[row.parts.part + row.parts.value]" @click="addPart(row)" />
              </UTooltip>
            </div>
            <div v-else>

              <UTooltip text="Create this part and add it to this project">
                <UButton class="mr-2" label=""
                  :icon="creating[row.parts.part + row.parts.value] ? '' : 'i-heroicons-outline-plus'"
                  @click="createPart(row)" :loading="creating[row.parts.part + row.parts.value]" />
              </UTooltip>
            </div>
          </div>
        </template>

        <template #parts.part-data="{ row }">
          <div v-if="row.parts.id">
            <UButton class="p-0" variant="link" @click="editPart(row.parts)">
              {{ row.parts.part }}
            </UButton>
            <UButton size="xs" class="text-xs px-0" variant="link" v-if="row.parts.parent"
              @click="editPart(row.parts.parent)">
              {{ row.parts.quantity_of }} × {{ row.parts.parent.part === row.parts.parent.value ? '' :
                row.parts.parent.part }} {{ row.parts.parent.value }}
            </UButton>
          </div>
          <div v-else>
            {{ row.parts.part }}
          </div>

          <div>
            <strong>
              {{ row.part }}
            </strong>
          </div>


        </template>

        <template #parts.value-data="{ row }">
          <UTooltip :text="row.parts.value">
            <UButton class="p-0 " v-if="row.parts.id" variant="link" @click="editPart(row.parts)">
              <div class="max-w-[100px] truncate overflow-hidden">
                {{ row.parts.value }}
              </div>
            </UButton>
            <div v-else class="max-w-[100px] truncate overflow-hidden">
              {{ row.parts.value }}
            </div>
          </UTooltip>
        </template>

        <template #parts.footprint-data="{ row }">
          <UTooltip :text="row.parts.footprint">
            <div class="max-w-[100px] truncate overflow-hidden">

              {{ row.parts.footprint }}

            </div>
          </UTooltip>
        </template>

        <template #quantity-data="{ row }">
          <div>
            {{ row.quantity }}
          </div>
        </template>

        <template #parts.quantity-data="{ row }">
          <PartsQuantity :part="row.parts" />
        </template>

        <template #locations-data="{ row }">
          <PartsLocation :part="row.parts" />
        </template>
      </UTable>
      <div class="text-right" v-if="newParts">
        <UButton @click="addAll">Add all</UButton>
      </div>
    </div>

    <div class="flex justify-end px-3 py-3.5 border-t border-gray-200 dark:border-gray-700">
      <UPagination v-if="pageCount < project.project_parts.length" v-model="page" :page-count="pageCount"
        :total="project.project_parts.length" />
    </div>
    <ProjectsPartQRCodeModal :open="qrModal" :projectPart="qrPart" @close="qrModal = false" />
    <ProjectsProjectPartModal :open="ppModal" :projectPart="ppPart" @close="ppModal = false" @refresh="emit('refresh')" />

    <PartsPartModal :partModal="partModal" :selectedPart="selectedPart" :saving="saving" @close="partModal = false"
      @save="savePart" @setParent="setParent" @refresh="refresh" @saved="partSaved" />
  </UCard>
</template>

<script lang="ts" setup>

const props = defineProps({
  project: {
    type: Object as Project,
    required: true,
  },
  parts: {
    type: Array<Part>,
    required: true,
  },
})

const toast = useToast()
const router = useRouter()
const client = useSupabaseClient()
const user = useSupabaseUser()
const emit = defineEmits(['refresh', 'setParts'])

const creating = ref({})
const qrModal = ref(false)
const selectedPart = ref({})
const saving = ref(false)
const partModal = ref(false)

const qrPart = ref({})
const adding = ref({})
const saveQty = ref({})
const projectParts = ref(props.project.project_parts)
const uploadModal = ref(false)
const deleting = ref(0)

const ppModal = ref(false)
const ppPart = ref({})

const newppModal = ref(false)
const newppPart = ref({ project_id: props.project.id })

const selected = ref([])
const deletingAll = ref(false)

const newParts = computed(() => {
  return projectParts.value.find(pp => !pp.id)
})

const page = ref(1)
const pageCount = 20

const sorted = computed(() => {
  return props.project.project_parts.sort((a: ProjectPart, b: ProjectPart) => {
    if (a.parts.part === b.parts.part) {
      return a.parts.value.localeCompare(b.parts.value)
    }
    return a.parts.part.localeCompare(b.parts.part)
  })
})

const rows = computed(() => {
  return sorted.value.slice((page.value - 1) * pageCount, (page.value) * pageCount)
})

const refresh = () => {
  emit('refresh')
}

const setParent = () => {
  partModal.value = false
  selectedPart.value = selectedPart.value.parent
  partModal.value = true
}

watch(
  () => props.project,
  () => {
    projectParts.value = props.project.project_parts;
  }
)

const deleteParts = async () => {
  deletingAll.value = true
  await client.from('project_parts').delete().in('id', selected.value.map(p => p.id))
  emit('refresh')
  deletingAll.value = false
}

const printTags = () => {
  const routeData = router.resolve({ path: `/projects/${props.project.id}/print`, query: { ids: selected.value.map(p => p.id) } });
  window.open(routeData.href, '_blank');
}


const printTag = (row: ProjectPart) => {
  qrPart.value = row
  qrModal.value = true
}

const saveQuantity = async (row: ProjectPart) => {
  saveQty.value[row.id] = true
  await client.from('project_parts').update({ quantity: row.quantity }).eq('id', row.id)
  saveQty.value[row.id] = false
}

const editPart = (part: Part) => {
  selectedPart.value = part
  partModal.value = true
}

const editProjectPart = (row) => {
  ppPart.value = row
  ppModal.value = true
}

const removeProjectPart = async (row) => {
  deleting.value = row.id
  await client.from('project_parts').delete().eq('id', row.id)
  projectParts.value = projectParts.value.filter((pp: ProjectPart) => pp.id !== row.id)
  refresh()
  deleting.value = 0
}

const addAll = () => {
  projectParts.value.forEach((pp: ProjectPart) => {
    if (!pp.id && pp.parts && pp.parts.id) {
      addPart(pp)
    } else if (!pp.id && !pp.parts.id) {
      createPart(pp)
    }
  })
}


const savePart = async () => {
  saving.value = true
  const p: Part = { ...selectedPart.value }

  delete p.locations
  if (p.id) {
    p.owner_id = user.value.id
    const r = await client.from('parts').update({ ...p }).eq('id', p.id).select(partFields())
      .eq('id', p.id)
    if (r.error) {
      toast.add({
        id: 'save_part_fail',
        title: `Could not update part ${p.part}.`,
        description: r.error.message,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 10000,
        color: 'red'
      })
    } else {
      partModal.value = false;

      refresh()
    }
    saving.value = false
  } else {
    p.owner_id = user.value.id
    delete p.id
    const r = await client.from('parts').insert({ ...p }).select(partFields())
    if (r.error) {
      toast.add({
        id: 'create_part_fail',
        title: `Could not create part ${p.part}.`,
        description: r.error.message,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 10000,
        color: 'red'
      })
    } else {
      partModal.value = false;
      const part = r.data[0]
      const pp: ProjectPart = {
        part_id: part.id,
        parts: part,
        project_id: props.project.id,
        quantity: 0,
      }
      const np = await addPart(pp)
      props.project.project_parts.push(np)

      refresh()
    }
  }
  saving.value = false
}

const partSaved = (part: Part) => {
  addPart({
    parts: part,
    quantity: 0,
    reference: 0,
  })
}

const addPart = async (row) => {
  adding.value[row.parts.part + row.parts.value] = true
  const r2 = await client.from('project_parts').insert({
    project_id: props.project.id,
    part_id: row.parts.id,
    quantity: row.quantity,
    references: row.references,
  }).select(`id, parts(${partFields()}), quantity`)

  if (r2.error) {
    toast.add({
      id: 'project_add_part_fail',
      title: `Could not add part ${row.parts.part}.`,
      description: r2.error.message,
      icon: 'i-heroicons-outline-exclamation-triangle',
      timeout: 10000,
      color: 'red'
    })
  } else {
    row.id = r2.data[0].id
    //refresh()
  }
  adding.value[row.parts.part + row.parts.value] = false
  return r2.data[0]
}

const createPart = async (row: ProjectPart) => {
  creating.value[row.parts.part + row.parts.value] = true

  const p = { ...row.parts }
  p.owner_id = user.value.id
  delete p.id
  const r = await client.from('parts').insert({ ...p }).select(partFields())

  if (r.error) {
    toast.add({
      id: 'create_part_fail',
      title: `Could not create part ${p.part}.`,
      description: r.error.message,
      icon: 'i-heroicons-outline-exclamation-triangle',
      timeout: 10000,
      color: 'red'
    })
  } else {
    const newPart = r.data[0]
    row.parts = newPart
    const r2 = await client.from('project_parts').insert({
      project_id: props.project.id,
      part_id: newPart.id,
      quantity: row.quantity,
      references: row.references,
    }).select()

    if (r2.error) {
      toast.add({
        id: 'project_add_part_fail',
        title: `Could not add part ${row.parts.part}.`,
        description: r2.error.message,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 10000,
        color: 'red'
      })
    } else {
      row.id = r2.data[0].id
      //refresh()
    }

  }
  creating.value[row.parts.part + row.parts.value] = false
}

const items = [
  [{
    label: 'KiCAD BOM',
    avatar: {
      src: 'https://user-images.githubusercontent.com/352202/53980744-60746100-4111-11e9-9f8c-17ca6b50efd8.png'
    },
    click: () => {
      uploadModal.value = true
    }
  }],
  [{
    label: 'Create part',
    icon: 'i-heroicons-outline-plus-circle',
    click: () => {
      selectedPart.value = {}
      partModal.value = true
    }
  }],
  [{
    label: 'Add existing part',
    icon: 'i-heroicons-outline-plus',
    click: () => {
      newppModal.value = true
    }
  }],
]

const columns = [
  {
    key: "parts.part",
    label: "Part",
    sortable: true,
  },
  {
    key: "parts.value",
    label: "Value",
    sortable: true,
  },
  {
    key: "parts.footprint",
    label: "Footprint",
    class: "max-w-xs",
    sortable: true,
  },
  {
    key: "parts.description",
    label: "Description",
    class: "max-w-md",
    sortable: true,
  },
  {
    key: "locations",
    label: "Locations",
    sortable: true,
  },
  {
    key: "quantity",
    label: "Quantity per PCB",
    sortable: true,
  },
  {
    key: "parts.quantity",
    label: "Available",
    sortable: true,
  },
  {
    key: "references",
    label: "References",
    sortable: true,
  },
  {
    key: "id",
    label: "Tools"
  },
]

const kicadCSV = async (files: Array<File>) => {
  const projectParts = await kicadImporter(files, props.parts, props.project)

  props.project.project_parts = projectParts
}

</script>

<style></style>