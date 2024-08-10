import { computed, inject, onMounted, provide, ref, shallowRef } from "vue";
import * as mfm from "mfm-js";
import type * as Misskey from "misskey-js";
import MkNoteSub from "@/components/MkNoteSub.vue";
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
import { notePage } from "@/filters/note.js";
import number from "@/filters/number.js";
import * as os from "@/os.js";
import { misskeyApi, misskeyApiGet } from "@/scripts/misskey-api.js";
import * as sound from "@/scripts/sound.js";
import { defaultStore, noteViewInterruptors } from "@/store.js";
import { reactionPicker } from "@/scripts/reaction-picker.js";
import { extractUrlFromMfm } from "@/scripts/extract-url-from-mfm.js";
import { $i } from "@/account.js";
import { i18n } from "@/i18n.js";
import { host } from "@/config.js";
import {
	getNoteClipMenu,
	getNoteMenu,
	getRenoteMenu,
} from "@/scripts/get-note-menu.js";
import { useNoteCapture } from "@/scripts/use-note-capture.js";
import { deepClone } from "@/scripts/clone.js";
import { useTooltip } from "@/scripts/use-tooltip.js";
import { claimAchievement } from "@/scripts/achievements.js";
import MkRippleEffect from "@/components/MkRippleEffect.vue";
import { showMovedDialog } from "@/scripts/show-moved-dialog.js";
import MkUserCardMini from "@/components/MkUserCardMini.vue";
import MkPagination, { type Paging } from "@/components/MkPagination.vue";
import MkReactionIcon from "@/components/MkReactionIcon.vue";
import MkButton from "@/components/MkButton.vue";
import { isEnabledUrlPreview } from "@/instance.js";
import type { Keymap } from "@/scripts/hotkey.js";

const props = withDefaults(
	defineProps<{
		note: Misskey.entities.Note;
		initialTab: string;
	}>(),
	{
		initialTab: "replies",
	},
);

const inChannel = inject("inChannel", null);

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
const isDeleted = ref(false);
const muted = ref(
	$i ? checkWordMute(appearNote.value, $i, $i.mutedWords) : false,
);
const translation = ref<Misskey.entities.NotesTranslateResponse | null>(null);
const translating = ref(false);
const parsed = appearNote.value.text ? mfm.parse(appearNote.value.text) : null;
const urls = parsed
	? extractUrlFromMfm(parsed).filter(
			(url) =>
				appearNote.value.renote?.url !== url &&
				appearNote.value.renote?.uri !== url,
		)
	: null;
const showTicker =
	defaultStore.state.instanceTicker === "always" ||
	(defaultStore.state.instanceTicker === "remote" &&
		appearNote.value.user.instance);
const conversation = ref<Misskey.entities.Note[]>([]);
const replies = ref<Misskey.entities.Note[]>([]);
const canRenote = computed(
	() =>
		["public", "home"].includes(appearNote.value.visibility) ||
		appearNote.value.userId === $i?.id,
);

const pleaseLoginContext = computed<OpenOnRemoteOptions>(() => ({
	type: "lookup",
	url: `https://${host}/notes/${appearNote.value.id}`,
}));

const keymap = {
	r: () => reply(),
	"e|a|plus": () => react(),
	q: () => renote(),
	m: () => showMenu(),
	c: () => {
		if (!defaultStore.state.showClipButtonInNoteFooter) return;
		clip();
	},
	o: () => galleryEl.value?.openGallery(),
	"v|enter": () => {
		if (appearNote.value.cw != null) {
			showContent.value = !showContent.value;
		}
	},
	esc: {
		allowRepeat: true,
		callback: () => blur(),
	},
} as const satisfies Keymap;

provide("react", (reaction: string) => {
	misskeyApi("notes/reactions/create", {
		noteId: appearNote.value.id,
		reaction: reaction,
	});
});

const tab = ref(props.initialTab);
const reactionTabType = ref<string | null>(null);

const renotesPagination = computed<Paging>(() => ({
	endpoint: "notes/renotes",
	limit: 10,
	params: {
		noteId: appearNote.value.id,
	},
}));

const reactionsPagination = computed<Paging>(() => ({
	endpoint: "notes/reactions",
	limit: 10,
	params: {
		noteId: appearNote.value.id,
		type: reactionTabType.value,
	},
}));

useNoteCapture({
	rootEl: rootEl,
	note: appearNote,
	pureNote: note,
	isDeletedRef: isDeleted,
});

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

function renote() {
	pleaseLogin(undefined, pleaseLoginContext.value);
	showMovedDialog();

	const { menu } = getRenoteMenu({ note: note.value, renoteButton });
	os.popupMenu(menu, renoteButton.value);
}

function reply(): void {
	pleaseLogin(undefined, pleaseLoginContext.value);
	showMovedDialog();
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
	const isLink = (el: HTMLElement): boolean => {
		if (el.tagName === "A") return true;
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
		});
		os.contextMenu(menu, ev).then(focus).finally(cleanup);
	}
}

function showMenu(): void {
	const { menu, cleanup } = getNoteMenu({
		note: note.value,
		translating,
		translation,
		isDeleted,
	});
	os.popupMenu(menu, menuButton.value).then(focus).finally(cleanup);
}

async function clip(): Promise<void> {
	os.popupMenu(
		await getNoteClipMenu({ note: note.value, isDeleted }),
		clipButton.value,
	).then(focus);
}

function showRenoteMenu(): void {
	if (!isMyRenote) return;
	pleaseLogin(undefined, pleaseLoginContext.value);
	os.popupMenu(
		[
			{
				text: i18n.ts.unrenote,
				icon: "ti ti-trash",
				danger: true,
				action: () => {
					misskeyApi("notes/delete", {
						noteId: note.value.id,
					});
					isDeleted.value = true;
				},
			},
		],
		renoteTime.value,
	);
}

function focus() {
	rootEl.value?.focus();
}

function blur() {
	rootEl.value?.blur();
}

const repliesLoaded = ref(false);

function loadReplies() {
	repliesLoaded.value = true;
	misskeyApi("notes/children", {
		noteId: appearNote.value.id,
		limit: 30,
	}).then((res) => {
		replies.value = res;
	});
}

const conversationLoaded = ref(false);

function loadConversation() {
	conversationLoaded.value = true;
	if (appearNote.value.replyId == null) return;
	misskeyApi("notes/conversation", {
		noteId: appearNote.value.replyId,
	}).then((res) => {
		conversation.value = res.reverse();
	});
}
