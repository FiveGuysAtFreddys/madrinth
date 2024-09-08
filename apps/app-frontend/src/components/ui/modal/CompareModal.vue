<template>
  <Modal class="scrollbar" ref="modal" :header="header" :noblur="!themeStore.advancedRendering" @hide="onModalHide">
    <div class="modal-body">
      <div class="dropdowns">
        <DropdownSelect v-model="selectedInstance" class="sort-dropdown" name="Instance Dropdown"
          :options="instanceOptions" placeholder="Select an instance..." />
        <DropdownSelect v-model="selectedProjectType" :options="Object.keys(selectableProjectTypes)" default-value="All"
          name="project-type-dropdown" color="primary" />
      </div>
      <div class="table" v-if="search.length > 0">
        <div v-for="mod in search" :key="mod.file_name" class="table-row">
          <div class="table-cell table-text name-cell">
            <div class="mod-content">
              <Avatar :src="mod.icon" />
              <span v-tooltip="`${mod.name}`" class="title">{{ mod.name }}</span>
            </div>
          </div>
          <div class="table-cell table-text manage">
            <Button v-tooltip="'Add to current modpack'" color="green" icon-only @click="console.log('hey')">
              <PlusIcon />
            </Button>
          </div>
        </div>
      </div>
      <p v-else>This modpack does not have any mods</p>
      <div class="button-group push-right">
        <Button color="green" @click="modal.hide()">
          <CheckIcon /> Finish
        </Button>
      </div>
    </div>
  </Modal>
</template>

<script setup>
import { computed, ref, shallowRef, watch } from 'vue'
import { Pagination, Avatar, Button, Modal, DropdownSelect } from '@modrinth/ui'
import { show_ads_window, hide_ads_window } from '@/helpers/ads.js'
import { useTheming } from '@/store/theme.js'
import { CheckIcon, PlusIcon } from '@modrinth/assets'
import {
  get_organization_many,
  get_project_many,
  get_team_many,
  get_version_many,
} from '@/helpers/cache.js'
import { handleError } from '@/store/notifications.js'
import { get_projects } from '@/helpers/profile.js'
import { list } from '@/helpers/profile.js'
import { formatProjectType } from '@modrinth/utils'

const themeStore = useTheming()

const props = defineProps({
  header: {
    type: String,
    default: null,
  },
  closable: {
    type: Boolean,
    default: true,
  },
  onHide: {
    type: Function,
    default() {
      return () => { }
    },
  },
  instance: {
    type: Object,
    default() {
      return {}
    },
  },
})

const modal = ref(null)

function onModalHide() {
  show_ads_window()
  props.onHide()
}

defineExpose({
  show: () => {
    hide_ads_window()
    modal.value.show()
  },
  hide: () => {
    onModalHide()
    modal.value.hide()
  },
})

const instances = shallowRef(await list().catch(handleError))
const curInstance = ref(instances.value[0])
const initialInstance = props.instance

const selectedInstance = ref(instances.value[0].name)
const selectedProjectType = ref('Mods')

const selectableProjectTypes = computed(() => {
  const obj = { All: 'all' }

  for (const project of projects.value) {
    obj[project.project_type ? formatProjectType(project.project_type) + 's' : 'Other'] =
      project.project_type
  }

  return obj
})

const instanceOptions = computed(() => instances.value.map((instance) => instance.name).filter((name) => { return initialInstance.name != name }))

watch(selectedInstance, (newInstanceName) => {
  const selected = instances.value.find((instance) => instance.name === newInstanceName)
  if (selected) {
    curInstance.value = selected
    initProjects()
  }
})


const projects = ref([])
const selectionMap = ref(new Map())
const initialProjects = ref([])

const initProjects = async (cacheBehaviour) => {
  const newProjects = []

  console.log(curInstance.value);

  const profileProjects = await get_projects(curInstance.value.path, cacheBehaviour)
  const fetchProjects = []
  const fetchVersions = []

  for (const value of Object.values(profileProjects)) {
    if (value.metadata) {
      fetchProjects.push(value.metadata.project_id)
      fetchVersions.push(value.metadata.version_id)
    }
  }

  const [modrinthProjects, modrinthVersions] = await Promise.all([
    await get_project_many(fetchProjects).catch(handleError),
    await get_version_many(fetchVersions).catch(handleError),
  ])

  const [modrinthTeams, modrinthOrganizations] = await Promise.all([
    await get_team_many(modrinthProjects.map((x) => x.team)).catch(handleError),
    await get_organization_many(
      modrinthProjects.map((x) => x.organization).filter((x) => !!x),
    ).catch(handleError),
  ])

  for (const [path, file] of Object.entries(profileProjects)) {
    if (file.metadata) {
      const project = modrinthProjects.find((x) => file.metadata.project_id === x.id)
      const version = modrinthVersions.find((x) => file.metadata.version_id === x.id)

      if (project && version) {
        const org = project.organization
          ? modrinthOrganizations.find((x) => x.id === project.organization)
          : null

        const team = modrinthTeams.find((x) => x[0].team_id === project.team)

        let owner

        if (org) {
          owner = org.name
        } else if (team) {
          owner = team.find((x) => x.is_owner).user.username
        } else {
          owner = null
        }

        newProjects.push({
          path,
          name: project.title,
          slug: project.slug,
          author: owner,
          version: version.version_number,
          file_name: file.file_name,
          icon: project.icon_url,
          disabled: file.file_name.endsWith('.disabled'),
          updateVersion: file.update_version_id,
          outdated: !!file.update_version_id,
          project_type: project.project_type,
          id: project.id,
        })
      }

      continue
    }

    newProjects.push({
      path,
      name: file.file_name.replace('.disabled', ''),
      author: '',
      version: null,
      file_name: file.file_name,
      icon: null,
      disabled: file.file_name.endsWith('.disabled'),
      outdated: false,
      project_type: file.project_type,
    })
  }

  projects.value = newProjects.filter(
    (newProject) => !initialProjects.value.some(
      (initialProject) => initialProject.name == newProject.name
    )
  );

  console.log(initialProjects.value.map((p) => p.name));
  console.log(projects.value.map((p) => p.name));

  const newSelectionMap = new Map()
  for (const project of projects.value) {
    newSelectionMap.set(
      project.path,
      selectionMap.value.get(project.path) ??
      selectionMap.value.get(project.path.slice(0, -9)) ??
      selectionMap.value.get(project.path + '.disabled') ??
      false,
    )
  }
  selectionMap.value = newSelectionMap
}
await initProjects()
initialProjects.value = [...projects.value];

const search = computed(() => {
  const projectType = selectableProjectTypes.value[selectedProjectType.value]
  return projects.value.filter((mod) => {
    return (
      projectType === 'all' || mod.project_type === projectType
    )
  })
})
</script>

<style scoped lang="scss">
.table {
  margin-block-start: 0;
  border-radius: var(--radius-lg);
  border: 2px solid var(--color-bg);
}

.modal-body {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: var(--gap-lg);

  .button-group {
    display: flex;
    justify-content: flex-end;
    gap: 0.5rem;
  }

  strong {
    color: var(--color-contrast);
  }
}

.table-row {
  grid-template-columns: 1fr min-content;

  .name-cell {
    padding-left: 1rem;
  }
}

.table-cell {
  align-items: center;
}

.card-row {
  display: flex;
  align-items: center;
  gap: var(--gap-md);
  justify-content: space-between;
  background-color: var(--color-raised-bg);
}

.mod-card {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  gap: var(--gap-sm);
  justify-content: flex-start;
  margin-bottom: 0.5rem;
  white-space: nowrap;
  align-items: center;

  :deep(.dropdown-row) {
    .btn {
      height: 2.5rem !important;
    }
  }

  :deep(.btn) {
    height: 2.5rem;
  }

  .dropdown-input {
    flex-grow: 1;

    .animated-dropdown {
      width: unset;

      :deep(.selected) {
        border-radius: var(--radius-md) 0 0 var(--radius-md);
      }
    }

    .iconified-input {
      width: 100%;

      input {
        flex-basis: unset;
      }
    }

    :deep(.animated-dropdown) {
      .render-down {
        border-radius: var(--radius-md) 0 0 var(--radius-md) !important;
      }

      .options-wrapper {
        margin-top: 0.25rem;
        width: unset;
        border-radius: var(--radius-md);
      }

      .options {
        border-radius: var(--radius-md);
        border: 1px solid var(--color);
      }
    }
  }
}

.list-card {
  margin-top: 0.5rem;
}

.text-combo {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.name-cell {
  padding-left: 0;

  .btn {
    margin-left: var(--gap-sm);
    min-width: unset;
  }
}

.dropdown {
  width: 7rem !important;
}

.sort {
  padding-left: 0.5rem;
}

.second-row {
  display: flex;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: var(--gap-sm);

  .chips {
    flex-grow: 1;
  }
}

.mod-content {
  display: flex;
  align-items: center;
  gap: 1rem;

  .mod-text {
    display: flex;
    flex-direction: column;
  }

  .title {
    color: var(--color-contrast);
    font-weight: bolder;
  }
}

.dropdowns {
  display: flex;
  gap: 10px;
}
</style>