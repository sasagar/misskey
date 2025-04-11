import { onMounted, ref } from "vue";
import * as os from "@/os.js";
import { prefer } from "@/preferences.js";

defineProps<{
	message: string;
}>();

const emit = defineEmits<(ev: "closed") => void>();

const zIndex = os.claimZIndex("high");
const showing = ref(true);

onMounted(() => {
	window.setTimeout(() => {
		showing.value = false;
	}, 4000);
});
