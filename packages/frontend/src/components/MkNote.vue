import {
	computed,
	inject,
	onMounted,
	ref,
	shallowRef,
	type Ref,
	watch,
	provide,
} from "vue";
import * as mfm from "mfm-js";
import type * as Misskey from "misskey-js";
import MkNoteSub from "@/components/MkNoteSub.vue";
import MkNoteHeader from "@/components/MkNoteHeader.vue";
import MkNoteSimple from "@/components/MkNoteSimple.vue";
import MkReactionsViewer from "@/components/MkReactionsViewer.vue";
import MkReactionsViewerDetails from "@/components/MkReactionsViewer.details.vue";
import type MkMediaList from "@/components/MkMediaList.vue";
import MkCwButton from "@/components/MkCwButton.vue";
import MkPoll from "@/components/MkPoll.vue";
import MkUsersTooltip from "@/components/MkUsersTooltip.vue";
import MkUrlPreview from "@/components/MkUrlPreview.vue";
import MkInstanceTicker from "@/components/MkInstanceTicker.vue";
import {
	pleaseLogin,
	type OpenOnRemoteOptions,
} from "@/scripts/please-login.js";
import { checkWordMute } from "@/scripts/check-word-mute.js";
import { userPage } from "@/filters/user.js";
import number from "@/filters/number.js";
import * as os from "@/os.js";
import * as sound from "@/scripts/sound.js";
import { misskeyApi, misskeyApiGet } from "@/scripts/misskey-api.js";
import { defaultStore, noteViewInterruptors } from "@/store.js";
import { reactionPicker } from "@/scripts/reaction-picker.js";
import { extractUrlFromMfm } from "@/scripts/extract-url-from-mfm.js";
import { $i } from "@/account.js";
import { i18n } from "@/i18n.js";
import {
	getAbuseNoteMenu,
	getCopyNoteLinkMenu,
	getNoteClipMenu,
	getNoteMenu,
	getRenoteMenu,
} from "@/scripts/get-note-menu.js";
import { useNoteCapture } from "@/scripts/use-note-capture.js";
import { deepClone } from "@/scripts/clone.js";
import { useTooltip } from "@/scripts/use-tooltip.js";
import { claimAchievement } from "@/scripts/achievements.js";
import { getNoteSummary } from "@/scripts/get-note-summary.js";
import type { MenuItem } from "@/types/menu.js";
import MkRippleEffect from "@/components/MkRippleEffect.vue";
import { showMovedDialog } from "@/scripts/show-moved-dialog.js";
import { shouldCollapsed } from "@/scripts/collapsed.js";
import { host } from "@/config.js";
import { isEnabledUrlPreview } from "@/instance.js";
import type { Keymap } from "@/scripts/hotkey.js";
import { focusPrev, focusNext } from "@/scripts/focus.js";

const props = withDefaults(
	defineProps<{
		note: Misskey.entities.Note;
		pinned?: boolean;
		mock?: boolean;
		withHardMute?: boolean;
	}>(),
	{
		mock: false,
	},
);

provide("mock", props.mock);

const emit = defineEmits<{
	(ev: "reaction", emoji: string): void;
	(ev: "removeReaction", emoji: string): void;
}>();

const inTimeline = inject<boolean>("inTimeline", false);
const inChannel = inject("inChannel", null);
const currentClip = inject<Ref<Misskey.entities.Clip> | null>(
	"currentClip",
	null,
);

const note = ref(deepClone(props.note));

// plugin
if (noteViewInterruptors.length > 0) {
	onMounted(async () => {
		let result: Misskey.entities.Note | null = deepClone(note.value);
		for (const interruptor of noteViewInterruptors) {
			try {
				result = (await interruptor.handler(
					result!,
				)) as Misskey.entities.Note | null;
				if (result === null) {
					isDeleted.value = true;
					return;
				}
			} catch (err) {
				console.error(err);
			}
		}
		note.value = result as Misskey.entities.Note;
	});
}

const isRenote =
	note.value.renote != null &&
	note.value.reply == null &&
	note.value.text == null &&
	note.value.cw == null &&
	note.value.fileIds &&
	note.value.fileIds.length === 0 &&
	note.value.poll == null;

const rootEl = shallowRef<HTMLElement>();
const menuButton = shallowRef<HTMLElement>();
const renoteButton = shallowRef<HTMLElement>();
const renoteTime = shallowRef<HTMLElement>();
const reactButton = shallowRef<HTMLElement>();
const clipButton = shallowRef<HTMLElement>();
const appearNote = computed(() =>
	isRenote ? (note.value.renote as Misskey.entities.Note) : note.value,
);
const galleryEl = shallowRef<InstanceType<typeof MkMediaList>>();
const isMyRenote = $i && $i.id === note.value.userId;
const showContent = ref(false);
const parsed = computed(() =>
	appearNote.value.text ? mfm.parse(appearNote.value.text) : null,
);
const urls = computed(() =>
	parsed.value
		? extractUrlFromMfm(parsed.value).filter(
				(url) =>
					appearNote.value.renote?.url !== url &&
					appearNote.value.renote?.uri !== url,
			)
		: null,
);
const isLong = shouldCollapsed(appearNote.value, urls.value ?? []);
const collapsed = ref(appearNote.value.cw == null && isLong);
const isDeleted = ref(false);
const muted = ref(checkMute(appearNote.value, $i?.mutedWords));
const hardMuted = ref(
	props.withHardMute && checkMute(appearNote.value, $i?.hardMutedWords, true),
);
const translation = ref<Misskey.entities.NotesTranslateResponse | null>(null);
const translating = ref(false);
const showTicker =
	defaultStore.state.instanceTicker === "always" ||
	(defaultStore.state.instanceTicker === "remote" &&
		appearNote.value.user.instance);
const canRenote = computed(
	() =>
		["public", "home"].includes(appearNote.value.visibility) ||
		(appearNote.value.visibility === "followers" &&
			appearNote.value.userId === $i?.id),
);
const renoteCollapsed = ref(
	defaultStore.state.collapseRenotes &&
		isRenote &&
		(($i &&
			($i.id === note.value.userId || $i.id === appearNote.value.userId)) || // `||` must be `||`! See https://github.com/misskey-dev/misskey/issues/13131
			appearNote.value.myReaction != null),
);

const pleaseLoginContext = computed<OpenOnRemoteOptions>(() => ({
	type: "lookup",
	url: `https://${host}/notes/${appearNote.value.id}`,
}));

/* Overload FunctionにLintが対応していないのでコメントアウト
function checkMute(noteToCheck: Misskey.entities.Note, mutedWords: Array<string | string[]> | undefined | null, checkOnly: true): boolean;
function checkMute(noteToCheck: Misskey.entities.Note, mutedWords: Array<string | string[]> | undefined | null, checkOnly: false): boolean | 'sensitiveMute';
*/
function checkMute(
	noteToCheck: Misskey.entities.Note,
	mutedWords: Array<string | string[]> | undefined | null,
	checkOnly = false,
): boolean | "sensitiveMute" {
	if (mutedWords == null) return false;

	if (checkWordMute(noteToCheck, $i, mutedWords)) return true;
	if (noteToCheck.reply && checkWordMute(noteToCheck.reply, $i, mutedWords))
		return true;
	if (noteToCheck.renote && checkWordMute(noteToCheck.renote, $i, mutedWords))
		return true;

	if (checkOnly) return false;

	if (
		inTimeline &&
		!defaultStore.state.tl.filter.withSensitive &&
		noteToCheck.files?.some((v) => v.isSensitive)
	)
		return "sensitiveMute";
	return false;
}

const keymap = {
	r: () => {
		if (renoteCollapsed.value) return;
		reply();
	},
	"e|a|plus": () => {
		if (renoteCollapsed.value) return;
		react();
	},
	q: () => {
		if (renoteCollapsed.value) return;
		renote();
	},
	m: () => {
		if (renoteCollapsed.value) return;
		showMenu();
	},
	c: () => {
		if (renoteCollapsed.value) return;
		if (!defaultStore.state.showClipButtonInNoteFooter) return;
		clip();
	},
	o: () => {
		if (renoteCollapsed.value) return;
		galleryEl.value?.openGallery();
	},
	"v|enter": () => {
		if (renoteCollapsed.value) {
			renoteCollapsed.value = false;
		} else if (appearNote.value.cw != null) {
			showContent.value = !showContent.value;
		} else if (isLong) {
			collapsed.value = !collapsed.value;
		}
	},
	esc: {
		allowRepeat: true,
		callback: () => blur(),
	},
	"up|k|shift+tab": {
		allowRepeat: true,
		callback: () => focusBefore(),
	},
	"down|j|tab": {
		allowRepeat: true,
		callback: () => focusAfter(),
	},
} as const satisfies Keymap;

provide("react", (reaction: string) => {
	misskeyApi("notes/reactions/create", {
		noteId: appearNote.value.id,
		reaction: reaction,
	});
});

if (props.mock) {
	watch(
		() => props.note,
		(to) => {
			note.value = deepClone(to);
		},
		{ deep: true },
	);
} else {
	useNoteCapture({
		rootEl: rootEl,
		note: appearNote,
		pureNote: note,
		isDeletedRef: isDeleted,
	});
}

if (!props.mock) {
	useTooltip(renoteButton, async (showing) => {
		const renotes = await misskeyApi("notes/renotes", {
			noteId: appearNote.value.id,
			limit: 11,
		});

		const users = renotes.map((x) => x.user);

		if (users.length < 1) return;

		const { dispose } = os.popup(
			MkUsersTooltip,
			{
				showing,
				users,
				count: appearNote.value.renoteCount,
				targetElement: renoteButton.value,
			},
			{
				closed: () => dispose(),
			},
		);
	});

	if (appearNote.value.reactionAcceptance === "likeOnly") {
		useTooltip(reactButton, async (showing) => {
			const reactions = await misskeyApiGet("notes/reactions", {
				noteId: appearNote.value.id,
				limit: 10,
				_cacheKey_: appearNote.value.reactionCount,
			});

			const users = reactions.map((x) => x.user);

			if (users.length < 1) return;

			const { dispose } = os.popup(
				MkReactionsViewerDetails,
				{
					showing,
					reaction: "❤️",
					users,
					count: appearNote.value.reactionCount,
					targetElement: reactButton.value!,
				},
				{
					closed: () => dispose(),
				},
			);
		});
	}
}

function renote(viaKeyboard = false) {
	pleaseLogin(undefined, pleaseLoginContext.value);
	showMovedDialog();

	const { menu } = getRenoteMenu({
		note: note.value,
		renoteButton,
		mock: props.mock,
	});
	os.popupMenu(menu, renoteButton.value, {
		viaKeyboard,
	});
}

function reply(): void {
	pleaseLogin(undefined, pleaseLoginContext.value);
	if (props.mock) {
		return;
	}
	os.post({
		reply: appearNote.value,
		channel: appearNote.value.channel,
	}).then(() => {
		focus();
	});
}

function react(): void {
	pleaseLogin(undefined, pleaseLoginContext.value);
	showMovedDialog();
	if (appearNote.value.reactionAcceptance === "likeOnly") {
		sound.playMisskeySfx("reaction");

		if (props.mock) {
			return;
		}

		misskeyApi("notes/reactions/create", {
			noteId: appearNote.value.id,
			reaction: "❤️",
		});
		const el = reactButton.value;
		if (el) {
			const rect = el.getBoundingClientRect();
			const x = rect.left + el.offsetWidth / 2;
			const y = rect.top + el.offsetHeight / 2;
			const { dispose } = os.popup(
				MkRippleEffect,
				{ x, y },
				{
					end: () => dispose(),
				},
			);
		}
	} else {
		blur();
		reactionPicker.show(
			reactButton.value ?? null,
			note.value,
			(reaction) => {
				sound.playMisskeySfx("reaction");

				if (props.mock) {
					emit("reaction", reaction);
					return;
				}

				misskeyApi("notes/reactions/create", {
					noteId: appearNote.value.id,
					reaction: reaction,
				});
				if (
					appearNote.value.text &&
					appearNote.value.text.length > 100 &&
					Date.now() - new Date(appearNote.value.createdAt).getTime() < 1000 * 3
				) {
					claimAchievement("reactWithoutRead");
				}
			},
			() => {
				focus();
			},
		);
	}
}

function undoReact(targetNote: Misskey.entities.Note): void {
	const oldReaction = targetNote.myReaction;
	if (!oldReaction) return;

	if (props.mock) {
		emit("removeReaction", oldReaction);
		return;
	}

	misskeyApi("notes/reactions/delete", {
		noteId: targetNote.id,
	});
}

function toggleReact() {
	if (appearNote.value.myReaction == null) {
		react();
	} else {
		undoReact(appearNote.value);
	}
}

function onContextmenu(ev: MouseEvent): void {
	if (props.mock) {
		return;
	}

	const isLink = (el: HTMLElement): boolean => {
		if (el.tagName === "A") return true;
		// 再生速度の選択などのために、Audio要素のコンテキストメニューはブラウザデフォルトとする。
		if (el.tagName === "AUDIO") return true;
		if (el.parentElement) {
			return isLink(el.parentElement);
		}
		return false;
	};

	if (ev.target && isLink(ev.target as HTMLElement)) return;
	if (window.getSelection()?.toString() !== "") return;

	if (defaultStore.state.useReactionPickerForContextMenu) {
		ev.preventDefault();
		react();
	} else {
		const { menu, cleanup } = getNoteMenu({
			note: note.value,
			translating,
			translation,
			isDeleted,
			currentClip: currentClip?.value,
		});
		os.contextMenu(menu, ev).then(focus).finally(cleanup);
	}
}

function showMenu(): void {
	if (props.mock) {
		return;
	}

	const { menu, cleanup } = getNoteMenu({
		note: note.value,
		translating,
		translation,
		isDeleted,
		currentClip: currentClip?.value,
	});
	os.popupMenu(menu, menuButton.value).then(focus).finally(cleanup);
}

async function clip(): Promise<void> {
	if (props.mock) {
		return;
	}

	os.popupMenu(
		await getNoteClipMenu({
			note: note.value,
			isDeleted,
			currentClip: currentClip?.value,
		}),
		clipButton.value,
	).then(focus);
}

function showRenoteMenu(): void {
	if (props.mock) {
		return;
	}

	function getUnrenote(): MenuItem {
		return {
			text: i18n.ts.unrenote,
			icon: "ti ti-trash",
			danger: true,
			action: () => {
				misskeyApi("notes/delete", {
					noteId: note.value.id,
				});
				isDeleted.value = true;
			},
		};
	}

	if (isMyRenote) {
		pleaseLogin(undefined, pleaseLoginContext.value);
		os.popupMenu(
			[
				getCopyNoteLinkMenu(note.value, i18n.ts.copyLinkRenote),
				{ type: "divider" },
				getUnrenote(),
			],
			renoteTime.value,
		);
	} else {
		os.popupMenu(
			[
				getCopyNoteLinkMenu(note.value, i18n.ts.copyLinkRenote),
				{ type: "divider" },
				getAbuseNoteMenu(note.value, i18n.ts.reportAbuseRenote),
				$i?.isModerator || $i?.isAdmin ? getUnrenote() : undefined,
			],
			renoteTime.value,
		);
	}
}

function focus() {
	rootEl.value?.focus();
}

function blur() {
	rootEl.value?.blur();
}

function focusBefore() {
	focusPrev(rootEl.value);
}

function focusAfter() {
	focusNext(rootEl.value);
}

function readPromo() {
	misskeyApi("promo/read", {
		noteId: appearNote.value.id,
	});
	isDeleted.value = true;
}

function emitUpdReaction(emoji: string, delta: number) {
	if (delta < 0) {
		emit("removeReaction", emoji);
	} else if (delta > 0) {
		emit("reaction", emoji);
	}
}
