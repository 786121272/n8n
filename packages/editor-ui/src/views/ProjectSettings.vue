<script lang="ts" setup>
import { computed, ref, watch, onBeforeMount, onMounted, nextTick } from 'vue';
import { useRouter } from 'vue-router';
import { deepCopy } from 'n8n-workflow';
import { N8nFormInput } from 'n8n-design-system';
import { useUsersStore } from '@/stores/users.store';
import type { IUser } from '@/Interface';
import { useI18n } from '@/composables/useI18n';
import { useProjectsStore } from '@/stores/projects.store';
import ProjectTabs from '@/components/Projects/ProjectTabs.vue';
import { type Project, type ProjectRelation, ProjectTypes } from '@/types/projects.types';
import { useToast } from '@/composables/useToast';
import { VIEWS } from '@/constants';
import ProjectDeleteDialog from '@/components/Projects/ProjectDeleteDialog.vue';
import ProjectRoleUpgradeDialog from '@/components/Projects/ProjectRoleUpgradeDialog.vue';
import { useRolesStore } from '@/stores/roles.store';
import type { ProjectRole } from '@/types/roles.types';
import { useCloudPlanStore } from '@/stores/cloudPlan.store';
import { useTelemetry } from '@/composables/useTelemetry';
import { useDocumentTitle } from '@/composables/useDocumentTitle';
import ResourceListHeader from '@/components/layouts/ResourceListHeader.vue';

type FormDataDiff = {
	name?: string;
	role?: ProjectRelation[];
	memberAdded?: ProjectRelation[];
	memberRemoved?: ProjectRelation[];
};

const usersStore = useUsersStore();
const locale = useI18n();
const projectsStore = useProjectsStore();
const rolesStore = useRolesStore();
const cloudPlanStore = useCloudPlanStore();
const toast = useToast();
const router = useRouter();
const telemetry = useTelemetry();
const documentTitle = useDocumentTitle();
const dialogVisible = ref(false);
const upgradeDialogVisible = ref(false);

const isDirty = ref(false);
const isValid = ref(false);
const formData = ref<Pick<Project, 'name' | 'relations'>>({
	name: '',
	relations: [],
});
const projectRoleTranslations = ref<{ [key: string]: string }>({
	'project:viewer': locale.baseText('projects.settings.role.viewer'),
	'project:editor': locale.baseText('projects.settings.role.editor'),
	'project:admin': locale.baseText('projects.settings.role.admin'),
});
const nameInput = ref<InstanceType<typeof N8nFormInput> | null>(null);

const usersList = computed(() =>
	usersStore.allUsers.filter((user: IUser) => {
		const isAlreadySharedWithUser = (formData.value.relations || []).find(
			(r: ProjectRelation) => r.id === user.id,
		);

		return !isAlreadySharedWithUser;
	}),
);

const projects = computed(() =>
	projectsStore.availableProjects.filter(
		(project) => project.id !== projectsStore.currentProjectId,
	),
);
const projectRoles = computed(() =>
	rolesStore.processedProjectRoles.map((role) => ({
		...role,
		name: projectRoleTranslations.value[role.role],
	})),
);
const firstLicensedRole = computed(() => projectRoles.value.find((role) => role.licensed)?.role);

const onAddMember = (userId: string) => {
	isDirty.value = true;
	const user = usersStore.usersById[userId];
	if (!user) return;

	const { id, firstName, lastName, email } = user;
	const relation = { id, firstName, lastName, email } as ProjectRelation;

	if (firstLicensedRole.value) {
		relation.role = firstLicensedRole.value;
	}

	formData.value.relations.push(relation);
};

const onRoleAction = (user: Partial<IUser>, role: string) => {
	isDirty.value = true;
	const index = formData.value.relations.findIndex((r: ProjectRelation) => r.id === user.id);
	if (role === 'remove') {
		formData.value.relations.splice(index, 1);
	} else {
		formData.value.relations[index].role = role as ProjectRole;
	}
};

const onNameInput = () => {
	isDirty.value = true;
};

const onCancel = () => {
	formData.value.relations = projectsStore.currentProject?.relations
		? deepCopy(projectsStore.currentProject.relations)
		: [];
	formData.value.name = projectsStore.currentProject?.name ?? '';
	isDirty.value = false;
};

const makeFormDataDiff = (): FormDataDiff => {
	const diff: FormDataDiff = {};
	if (!projectsStore.currentProject) {
		return diff;
	}

	if (formData.value.name !== projectsStore.currentProject.name) {
		diff.name = formData.value.name ?? '';
	}

	if (formData.value.relations.length !== projectsStore.currentProject.relations.length) {
		diff.memberAdded = formData.value.relations.filter(
			(r: ProjectRelation) => !projectsStore.currentProject?.relations.find((cr) => cr.id === r.id),
		);
		diff.memberRemoved = projectsStore.currentProject.relations.filter(
			(cr: ProjectRelation) => !formData.value.relations.find((r) => r.id === cr.id),
		);
	}

	diff.role = formData.value.relations.filter((r: ProjectRelation) => {
		const currentRelation = projectsStore.currentProject?.relations.find((cr) => cr.id === r.id);
		return currentRelation?.role !== r.role && !diff.memberAdded?.find((ar) => ar.id === r.id);
	});

	return diff;
};

const sendTelemetry = (diff: FormDataDiff) => {
	if (diff.name) {
		telemetry.track('User changed project name', {
			project_id: projectsStore.currentProject?.id,
			name: diff.name,
		});
	}

	if (diff.memberAdded) {
		diff.memberAdded.forEach((r) => {
			telemetry.track('User added member to project', {
				project_id: projectsStore.currentProject?.id,
				target_user_id: r.id,
				role: r.role,
			});
		});
	}

	if (diff.memberRemoved) {
		diff.memberRemoved.forEach((r) => {
			telemetry.track('User removed member from project', {
				project_id: projectsStore.currentProject?.id,
				target_user_id: r.id,
			});
		});
	}

	if (diff.role) {
		diff.role.forEach((r) => {
			telemetry.track('User changed member role on project', {
				project_id: projectsStore.currentProject?.id,
				target_user_id: r.id,
				role: r.role,
			});
		});
	}
};

const onSubmit = async () => {
	try {
		if (isDirty.value && projectsStore.currentProject) {
			const diff = makeFormDataDiff();

			await projectsStore.updateProject({
				id: projectsStore.currentProject.id,
				name: formData.value.name,
				relations: formData.value.relations.map((r: ProjectRelation) => ({
					userId: r.id,
					role: r.role,
				})),
			});
			sendTelemetry(diff);
			isDirty.value = false;
			toast.showMessage({
				title: locale.baseText('projects.settings.save.successful.title', {
					interpolate: { projectName: formData.value.name ?? '' },
				}),
				type: 'success',
			});
		}
	} catch (error) {
		toast.showError(error, locale.baseText('projects.settings.save.error.title'));
	}
};

const onDelete = async () => {
	await projectsStore.getAvailableProjects();
	dialogVisible.value = true;
};

const onConfirmDelete = async (transferId?: string) => {
	try {
		if (projectsStore.currentProject) {
			const projectName = projectsStore.currentProject?.name ?? '';
			await projectsStore.deleteProject(projectsStore.currentProject.id, transferId);
			await router.push({ name: VIEWS.HOMEPAGE });
			toast.showMessage({
				title: locale.baseText('projects.settings.delete.successful.title', {
					interpolate: { projectName },
				}),
				type: 'success',
			});
			dialogVisible.value = true;
		}
	} catch (error) {
		toast.showError(error, locale.baseText('projects.settings.delete.error.title'));
	}
};

const selectProjectNameIfMatchesDefault = () => {
	if (formData.value.name === locale.baseText('projects.settings.newProjectName')) {
		nameInput.value?.inputRef?.focus();
		nameInput.value?.inputRef?.select();
	}
};

watch(
	() => projectsStore.currentProject,
	async () => {
		formData.value.name = projectsStore.currentProject?.name ?? '';
		formData.value.relations = projectsStore.currentProject?.relations
			? deepCopy(projectsStore.currentProject.relations)
			: [];
		await nextTick();
		selectProjectNameIfMatchesDefault();
	},
	{ immediate: true },
);

const headerIcon = computed(() => {
	if (projectsStore.currentProject?.type === ProjectTypes.Personal) {
		return 'user';
	} else if (projectsStore.currentProject?.name) {
		return 'layer-group';
	} else {
		return 'home';
	}
});

const projectName = computed(() => {
	if (!projectsStore.currentProject) {
		return locale.baseText('projects.menu.home');
	} else if (projectsStore.currentProject.type === ProjectTypes.Personal) {
		return locale.baseText('projects.menu.personal');
	} else {
		return projectsStore.currentProject.name;
	}
});

onBeforeMount(async () => {
	await usersStore.fetchUsers();
});

onMounted(() => {
	documentTitle.set(locale.baseText('projects.settings'));
	selectProjectNameIfMatchesDefault();
});
</script>

<template>
	<div :class="$style.projectSettings">
		<div :class="$style.header">
			<ResourceListHeader :icon="headerIcon" data-test-id="list-layout-header">
				<template #title>
					{{ projectName }}
				</template>
			</ResourceListHeader>
			<ProjectTabs />
		</div>
		<form @submit.prevent="onSubmit">
			<fieldset>
				<label for="projectName">{{ locale.baseText('projects.settings.name') }}</label>
				<N8nFormInput
					id="projectName"
					ref="nameInput"
					v-model="formData.name"
					label=""
					type="text"
					name="name"
					required
					data-test-id="project-settings-name-input"
					@input="onNameInput"
					@validate="isValid = $event"
				/>
			</fieldset>
			<fieldset>
				<label for="projectMembers">{{
					locale.baseText('projects.settings.projectMembers')
				}}</label>
				<N8nUserSelect
					id="projectMembers"
					class="mb-s"
					size="large"
					:users="usersList"
					:current-user-id="usersStore.currentUser?.id"
					:placeholder="$locale.baseText('workflows.shareModal.select.placeholder')"
					data-test-id="project-members-select"
					@update:model-value="onAddMember"
				>
					<template #prefix>
						<N8nIcon icon="search" />
					</template>
				</N8nUserSelect>
				<N8nUsersList
					:actions="[]"
					:users="formData.relations"
					:current-user-id="usersStore.currentUser?.id"
					:delete-label="$locale.baseText('workflows.shareModal.list.delete')"
				>
					<template #actions="{ user }">
						<div :class="$style.buttons">
							<N8nSelect
								class="mr-2xs"
								:model-value="user?.role || projectRoles[0].role"
								size="small"
								data-test-id="projects-settings-user-role-select"
								@update:model-value="onRoleAction(user, $event)"
							>
								<N8nOption
									v-for="role in projectRoles"
									:key="role.role"
									:value="role.role"
									:label="role.name"
									:disabled="!role.licensed"
								>
									{{ role.name
									}}<span
										v-if="!role.licensed"
										:class="$style.upgrade"
										@click="upgradeDialogVisible = true"
									>
										&nbsp;-&nbsp;{{ locale.baseText('generic.upgrade') }}
									</span>
								</N8nOption>
							</N8nSelect>
							<N8nButton
								type="tertiary"
								native-type="button"
								square
								icon="trash"
								data-test-id="project-user-remove"
								@click="onRoleAction(user, 'remove')"
							/>
						</div>
					</template>
				</N8nUsersList>
			</fieldset>
			<fieldset :class="$style.buttons">
				<div>
					<small v-if="isDirty" class="mr-2xs">{{
						locale.baseText('projects.settings.message.unsavedChanges')
					}}</small>
					<N8nButton
						:disabled="!isDirty"
						type="secondary"
						native-type="button"
						class="mr-2xs"
						data-test-id="project-settings-cancel-button"
						@click.stop.prevent="onCancel"
						>{{ locale.baseText('projects.settings.button.cancel') }}</N8nButton
					>
				</div>
				<N8nButton
					:disabled="!isDirty || !isValid"
					type="primary"
					data-test-id="project-settings-save-button"
					>{{ locale.baseText('projects.settings.button.save') }}</N8nButton
				>
			</fieldset>
			<fieldset>
				<hr class="mb-2xl" />
				<h3 class="mb-xs">{{ locale.baseText('projects.settings.danger.title') }}</h3>
				<small>{{ locale.baseText('projects.settings.danger.message') }}</small>
				<br />
				<N8nButton
					type="tertiary"
					native-type="button"
					class="mt-s"
					data-test-id="project-settings-delete-button"
					@click.stop.prevent="onDelete"
					>{{ locale.baseText('projects.settings.danger.deleteProject') }}</N8nButton
				>
			</fieldset>
		</form>
		<ProjectDeleteDialog
			v-model="dialogVisible"
			:current-project="projectsStore.currentProject"
			:projects="projects"
			@confirm-delete="onConfirmDelete"
		/>
		<ProjectRoleUpgradeDialog
			v-model="upgradeDialogVisible"
			:limit="projectsStore.teamProjectsLimit"
			:plan-name="cloudPlanStore.currentPlanData?.displayName"
		/>
	</div>
</template>

<style lang="scss" module>
.projectSettings {
	display: grid;
	width: 100%;
	justify-items: center;
	grid-auto-rows: max-content;

	form {
		width: 100%;
		max-width: 1280px;
		padding: 0 var(--spacing-2xl);

		fieldset {
			padding-bottom: var(--spacing-2xl);

			label {
				display: block;
				margin-bottom: var(--spacing-xs);
				font-size: var(--font-size-xl);
			}
		}
	}
}

.header {
	width: 100%;
	max-width: 1280px;
	padding: var(--spacing-l) var(--spacing-2xl) 0;
}

.upgrade {
	cursor: pointer;
}

.buttons {
	display: flex;
	justify-content: flex-end;
	align-items: center;
}
</style>
