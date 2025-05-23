<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<div class="_gaps_m">
	<MkCodeEditor v-model="installThemeCode" lang="json5">
		<template #label>{{ i18n.ts._theme.code }}</template>
	</MkCodeEditor>

	<div class="_buttons">
		<MkButton :disabled="installThemeCode == null || installThemeCode.trim() === ''" inline @click="() => previewTheme(installThemeCode)"><i class="ti ti-eye"></i> {{ i18n.ts.preview }}</MkButton>
		<MkButton :disabled="installThemeCode == null || installThemeCode.trim() === ''" primary inline @click="() => install(installThemeCode)"><i class="ti ti-check"></i> {{ i18n.ts.install }}</MkButton>
	</div>
</div>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue';
import MkCodeEditor from '@/components/MkCodeEditor.vue';
import MkButton from '@/components/MkButton.vue';
import { parseThemeCode, previewTheme, installTheme } from '@/theme.js';
import * as os from '@/os.js';
import { i18n } from '@/i18n.js';
import { definePage } from '@/page.js';
import { useRouter } from '@/router.js';

const router = useRouter();
const installThemeCode = ref<string | null>(null);

async function install(code: string): Promise<void> {
	try {
		const theme = parseThemeCode(code);
		await installTheme(code);
		os.alert({
			type: 'success',
			text: i18n.tsx._theme.installed({ name: theme.name }),
		});
		installThemeCode.value = null;
		router.push('/settings/theme');
	} catch (err) {
		switch (err.message.toLowerCase()) {
			case 'this theme is already installed':
				os.alert({
					type: 'info',
					text: i18n.ts._theme.alreadyInstalled,
				});
				break;

			default:
				os.alert({
					type: 'error',
					text: i18n.ts._theme.invalid,
				});
				break;
		}
		console.error(err);
	}
}

const headerActions = computed(() => []);

const headerTabs = computed(() => []);

definePage(() => ({
	title: i18n.ts._theme.install,
	icon: 'ti ti-download',
}));
</script>
