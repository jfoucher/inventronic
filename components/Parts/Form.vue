<template>
  <UForm class="space-y-4" :validate="validate" @submit="save" :state="state">
    <UCard :class="modal ? '' : 'm-2 md:m-8'" :ui="{ ring: '', divide: 'divide-y divide-gray-100 dark:divide-gray-800' }">
      <template #header>
        <div class="flex items-center justify-between">
          <h2 v-if="selectedPart.id">
            <UTooltip text="Print label">
              <UButton class="mr-8" icon="i-heroicons-qr-code" @click="printTag" />
            </UTooltip>

            Editing <strong>{{ part_type.label }} {{ selectedPart.value }}</strong>
          </h2>
          <h2 v-else>Create new part</h2>
          <UButton v-if="modal" color="gray" variant="ghost" icon="i-heroicons-x-mark-20-solid" class="-my-1"
            @click="emit('close')" />
        </div>
      </template>


      <div class="md:grid md:grid-cols-2 md:gap-x-8 md:gap-y-4">
        <UFormGroup label="Part of" name="parent_id">
          <USelectMenu v-model="state.parent" :options="parts.sort((a, b) => a.part.localeCompare(b.part))" searchable
            :uiMenu="{ option: { container: 'w-full block' } }">
            <template #label>
              <span v-if="state.parent && state.parent.part" class="truncate">
                {{ state.parent.part === state.parent.value ? state.parent.part :
                  state.parent.part + ' ' + state.parent.value }}

              </span>
              <span v-else>None</span>
            </template>
            <template #option="{ option: part, selected: sel }">
              <div class="flex flex-row justify-between px-1">
                <div class="grow">
                  <div class="flex flex-row justify-between">
                    <div class="truncate grow">
                      {{ part.part === part.value ? part.part : part.part + ' ' + part.value }}
                    </div>


                    <div v-if="part.location_parts && part.location_parts.length > 0"
                      class="truncate shrink text-slate-600 dark:text-slate-200">
                      <UBadge color="white" v-for="lp in part.location_parts" size="sm" class="text-xs">
                        {{ lp.locations.name }}
                      </UBadge>
                    </div>

                    <div v-else class="truncate shrink text-slate-600 dark:text-slate-200"></div>
                  </div>
                  <div class="text-xs text-slate-400 ">
                    {{ part.footprint }}
                  </div>
                </div>


                <div class="w-5 pe-2" v-if="!sel"></div>
              </div>
            </template>
          </USelectMenu>
        </UFormGroup>
        <UFormGroup label="Quantity of" name="qty_of">
          <UInput type="number" step="0.05" v-model=state.quantity_of color="white" variant="outline"
            placeholder="Quantity of referenced part" />
        </UFormGroup>
        <UFormGroup label="Part" name="part">
          <USelectMenu v-model="state.part" :options="part_types" searchable creatable>
          </USelectMenu>
        </UFormGroup>
        <UFormGroup label="Value" name="value">
          <UInput v-model=state.value color="white" variant="outline" placeholder="Part value" />
        </UFormGroup>
        <UFormGroup label="Description" name="description">
          <UInput v-model=state.description color="white" variant="outline" placeholder="Description" />
        </UFormGroup>
        <UFormGroup label="Footprint" name="footprint">
          <UInput v-model=state.footprint color="white" variant="outline" placeholder="Footprint" />
        </UFormGroup>

        <UFormGroup label="Minimum quantity" name="min_quantity">
          <UInput v-model=state.min_quantity color="white" variant="outline" placeholder="0" />
        </UFormGroup>
        <UFormGroup label="Price" name="price">
          <UInput v-model=state.price step="0.01" color="white" type="number" variant="outline" placeholder="" />
        </UFormGroup>
        <UFormGroup label="Ordering URL" name="ordering_url">
          <UInput v-model=state.ordering_url color="white" variant="outline" placeholder="" />
        </UFormGroup>

      </div>
      <div v-if="state.parent && state.parent.id" class="mt-4">
        <h2>Locations</h2>
        <UTable v-if="state.parent && state.parent.location_parts && state.parent.location_parts.length"
          :rows="[...state.parent.location_parts.map((lp: LocationPart) => { return { name: lp.locations.name, quantity: lp.quantity + ' ' + state.parent.part + ' ' + state.parent.value } }), { name: 'TOTAL', quantity: state.parent.location_parts.reduce((acc: number, lp: LocationPart) => acc + lp.quantity, 0) }]" />
        <div v-else class="text-center py-4 text-slate-400 text-sm">
          No stock.
          <UButton class="px-0" variant="link" label="Edit parent part" @click="setParent" />
          to add stock at selected locations
        </div>
      </div>
      <div v-else class="mt-4">
        <UFormGroup label="Locations" name="locations">
          <div v-for="lp in selectedLocations" class="flex flex-row items-end mb-4">
            <UFormGroup label="Location" class="grow">
              <USelectMenu v-model="lp.locations" :options="locations">
                <template #label>
                  {{ lp.locations.name ? lp.locations.name : 'None' }}
                </template>
                <template #option="{ option: location }">
                  {{ location.name }}
                </template>
              </USelectMenu>
            </UFormGroup>
            <UFormGroup label="Quantity" class="grow ml-4">
              <UInput type="number" step="0.05" v-model="lp.quantity" />
            </UFormGroup>
            <UButton class="shrink ml-4 h-8 w-8 mb-1" icon="i-heroicons-outline-trash" color="red"
              @click="deleteLocation(lp)" />
          </div>


        </UFormGroup>

        <UButton icon="i-heroicons-outline-plus" label="Add location"
          @click="selectedLocations.push({ key: Math.random() * 1000000, quantity: 0, locations: {} })" />
      </div>

      <PartsQRCodeModal :open="qrModal" :part="qrPart" @close="qrModal = false" />


      <template #footer>
        <UButton class="mr-4" type="submit" :loading="saving">
          <span v-if="saving">Saving...</span>
          <span v-else>Save</span>
        </UButton>
        <span v-if="modal !== false">or</span>
        <UButton v-if="modal !== false" variant="link" color="white" class="ml-0" label="Cancel"
          @click="$emit('close')" />
      </template>
    </UCard>
  </UForm>
</template>

<script lang="ts" setup>
import type { FormError } from '@nuxt/ui/dist/runtime/types';


const toast = useToast()

const user = useSupabaseUser()
const client = useSupabaseClient()
const emit = defineEmits()


const props = defineProps({
  selectedPart: {
    type: Object as Part,
    required: true,
  },
  modal: {
    type: Boolean,
    required: false,
  },
});



const saving = ref(false)
const qrModal = ref(false)
const qrPart = ref({})
const part_type = ref({ label: props.selectedPart.part ? props.selectedPart.part : 'Select or create' })

const { data: parts } = await useAsyncData('parts', async () => {
  const { data } = await client.from('parts').select(partFields()).order('created_at')

  return data
})

const state = reactive({
  part: props.selectedPart.part,
  value: props.selectedPart.value,
  description: props.selectedPart.description,
  footprint: props.selectedPart.footprint,
  min_quantity: props.selectedPart.min_quantity,
  price: props.selectedPart.price,
  ordering_url: props.selectedPart.ordering_url,
  parent: props.selectedPart.parent,
  location_parts: props.selectedPart.location_parts,
})

const setParent = () => {
  emit('setParent')
}

const printTag = () => {
  qrPart.value = props.selectedPart
  qrModal.value = true
}

watch(
  () => props.selectedPart,
  () => { part_type.value = { label: props.selectedPart.part } }
)

const { data: locations } = await useAsyncData('locations', async () => {
  const { data } = await client.from('locations').select().order('created_at')

  return data
})

const selectedLocations = ref(state.location_parts ? state.location_parts.map((lp: LocationPart) => {
  return { id: lp.id, quantity: lp.quantity, locations: lp.locations }
}) : [])

const part_types = computed(() => {
  const v = parts.value.map((p: Part) => p.part)
  const set = [...new Set(v)]
  return set.sort((a, b) => a.localeCompare(b)).map(l => { return { label: l } })
})

const validate = (state: any): FormError[] => {
  const errors = []
  if (!state.part) errors.push({ path: 'part', message: 'Required' })
  if (!state.value) errors.push({ path: 'value', message: 'Required' })
  return errors
}

const deleteLocation = async (lp: LocationPart) => {
  if (lp.id) {
    selectedLocations.value = selectedLocations.value.filter(ll => lp.id !== ll.id)
  } else if (lp.key) {
    selectedLocations.value = selectedLocations.value.filter(ll => lp.key !== ll.key)
  }
}


const save = async () => {
  saving.value = true
  const p = { ...state }

  delete p.location_parts
  delete p.project_parts
  delete p.quantity

  if (p.parent && p.parent.id) {
    p.parent_id = p.parent.id
  }

  delete p.parent

  p.part = p.part.label
  let part_id = props.selectedPart.id


  if (part_id) {
    p.owner_id = user.value.id
    const r = await client.from('parts').update({ ...p }).eq('id', part_id).select(partFields())
    if (r.error) {
      toast.add({
        id: 'part_save_error',
        title: 'Part save failed.',
        description: r.error.message,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 4000,
        color: 'red'
      })
      saving.value = false
      return
    }
  } else {
    p.owner_id = user.value.id
    delete p.id
    const r = await client.from('parts').insert({ ...p }).select(partFields())
    if (r.error) {
      toast.add({
        id: 'part_save_error',
        title: 'Could not save part.',
        description: r.error.message,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 4000,
        color: 'red'
      })
      saving.value = false
      return;
    } else {
      part_id = r.data[0].id
      emit('saved', r.data[0])
    }
  }

  // Delete all locations_parts for this part

  await client.from('location_parts').delete().eq('part_id', part_id)
  // Create of update location_parts
  for await (const loc of selectedLocations.value) {
    const r = await client.from('location_parts').insert({ part_id: part_id, location_id: loc.locations.id, quantity: loc.quantity })
    if (r.error) {
      let msg = r.error.message
      if (r.error.code === "23505") {
        msg = 'This part already exists at this location'
      } else if (r.error.code === "23502") {
        msg = 'Please choose a valid location for this part'
      }
      toast.add({
        id: 'part_save_error_duplicate' + loc.locations.id,
        title: 'Could not save part location.',
        description: msg,
        icon: 'i-heroicons-outline-exclamation-triangle',
        timeout: 10000,
        color: 'red'
      })
    }
  }
  saving.value = false
  emit('close')
  emit('refresh')

}

</script>

<style></style>