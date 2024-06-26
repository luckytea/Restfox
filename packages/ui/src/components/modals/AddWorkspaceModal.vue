<template>
    <form @submit.prevent="createWorkspace" v-if="showModalComp">
        <modal :title="!workspace ? 'New Workspace' : 'Workspace Properties'" v-model="showModalComp">
            <label>
                <div style="font-weight: 500; margin-bottom: var(--label-margin-bottom)">Name</div>
                <input type="text" class="full-width-input" v-model="workspaceName" :placeholder="!workspace ? 'New Workspace' : 'Workspace Name'" required spellcheck="false" v-focus>
            </label>

            <template v-if="isElectron">
                <div style="margin-top: 1rem">
                    <label>
                        <div style="font-weight: 500; margin-bottom: var(--label-margin-bottom)">Type</div>
                        <select class="full-width-input" v-model="workspaceType" :disabled="workspace">
                            <option v-for="workspaceTypeItem in workspaceTypes" :value="workspaceTypeItem.value">{{ workspaceTypeItem.label }}</option>
                        </select>
                    </label>
                </div>

                <div style="margin-top: 1rem" v-if="workspaceType === 'file'">
                    <label>
                        <div style="font-weight: 500; margin-bottom: var(--label-margin-bottom)">Choose a folder</div>
                        <div style="display: flex;">
                            <input type="text" class="full-width-input" v-model="workspaceLocation" placeholder="Folder Path" required spellcheck="false">
                            <button class="button" type="button" @click="openFolderDialog">Choose</button>
                        </div>
                    </label>
                    <div style="margin-top: 0.5rem">Provide an empty folder or select a folder with an existing workspace</div>
                </div>
            </template>

            <template #footer>
                <button class="button" v-if="!workspace">Create</button>
                <button class="button" v-else>Update</button>
            </template>
        </modal>
    </form>
</template>

<script>
import Modal from '@/components/Modal.vue'

export default {
    directives: {
        focus: {
            mounted(element, binding) {
                element.focus()
                if(!binding.instance.workspace) {
                    element.select()
                }
            }
        }
    },
    props: {
        showModal: Boolean,
        workspace: {
            type: Object,
            required: false
        },
        isElectron: {
            type: Boolean,
            default: false
        },
    },
    components: {
        Modal
    },
    data() {
        return {
            workspaceName: 'New Workspace',
            workspaceType: 'local',
            workspaceLocation: '',
            workspaceTypes: [
                { label: 'In Filesystem (Git Friendly)', value: 'file' },
                { label: 'In Application', value: 'local' }
            ]
        }
    },
    computed: {
        showModalComp: {
            get() {
                return this.showModal
            },
            set(value) {
                this.$emit('update:showModal', value)
            }
        }
    },
    watch: {
        workspace() {
            if(this.workspace) {
                this.workspaceName = this.workspace.name
                this.workspaceType = this.workspace._type ?? 'local'
                if(this.workspace._type === 'file') {
                    this.workspaceLocation = this.workspace.location
                } else {
                    this.workspaceLocation = ''
                }
            }
        },
        showModal() {
            if(this.showModal && this.workspace === undefined) {
                this.workspaceName = 'New Workspace'
                if(this.isElectron) {
                    this.workspaceType = 'file'
                } else {
                    this.workspaceType = 'local'
                }
                this.workspaceLocation = ''
            }
        },
        async workspaceLocation() {
            if(this.workspaceLocation) {
                this.getSetWorkspaceName()
            }
        }
    },
    methods: {
        async createWorkspace() {
            if(!this.workspace) {
                if(this.workspaceType === 'file') {
                    this.$store.dispatch('createWorkspace', {
                        name: this.workspaceName,
                        _type: this.workspaceType,
                        location: this.workspaceLocation
                    })
                } else {
                    this.$store.dispatch('createWorkspace', {
                        name: this.workspaceName
                    })
                }
            } else {
                if (this.workspaceType === 'file') {
                    this.$store.dispatch('updateWorkspace', {
                        _id: this.workspace._id,
                        updatedFields: {
                            name: this.workspaceName,
                            location: this.workspaceLocation
                        }
                    })
                } else {
                    this.$store.dispatch('updateWorkspace', {
                        _id: this.workspace._id,
                        updatedFields: {
                            name: this.workspaceName
                        }
                    })
                }
            }
            this.showModalComp = false
        },
        async getSetWorkspaceName() {
            try {
                const workspace = await window.electronIPC.getWorkspaceAtLocation(this.workspaceLocation)
                this.workspaceName = workspace.name
            } catch {}
        },
        async openFolderDialog() {
            this.workspaceLocation = await window.electronIPC.openFolderSelectionDialog()
            if (this.workspaceLocation) {
                this.getSetWorkspaceName()
            }
        }
    }
}
</script>
