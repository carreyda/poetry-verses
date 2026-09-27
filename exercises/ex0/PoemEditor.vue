<script setup lang="ts">
import { computed, onMounted, ref, watch } from "vue";

interface Props {
  poemId?: string;
  initialTitle?: string;
  maxLines?: number;
  readonly?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  poemId: undefined,
  initialTitle: "",
  maxLines: 12,
  readonly: false,
});

const emit = defineEmits<{
  save: [payload: { title: string; body: string }];
  cancel: [];
}>();

const title = ref(props.initialTitle);
const body = ref("");
const loading = ref(false);
const saving = ref(false);
const errorMsg = ref("");
const saved = ref({ title: props.initialTitle, body: "" });

const lines = computed(() =>
  body.value.split("\n").filter((line) => line.trim() !== ""),
);
const charCount = computed(() => body.value.replace(/\s/g, "").length);
const overLimit = computed(() => lines.value.length > props.maxLines);
const dirty = computed(
  () => saved.value.title !== title.value || saved.value.body !== body.value,
);
const canSubmit = computed(
  () =>
    !props.readonly &&
    !loading.value &&
    !saving.value &&
    title.value.trim() !== "" &&
    lines.value.length > 0 &&
    !overLimit.value,
);

async function fetchPoem(id: string): Promise<{ title: string; body: string }> {
  await new Promise((resolve) => setTimeout(resolve, 300));
  if (id === "missing") throw new Error("not found");
  return {
    title: "静夜思",
    body: "床前明月光，\n疑是地上霜。\n举头望明月，\n低头思故乡。",
  };
}

async function load() {
  if (!props.poemId) return;
  loading.value = true;
  errorMsg.value = "";
  try {
    const draft = await fetchPoem(props.poemId);
    title.value = draft.title;
    body.value = draft.body;
    saved.value = { ...draft };
  } catch {
    errorMsg.value = "加载诗歌失败，请稍后重试";
  } finally {
    loading.value = false;
  }
}

watch(() => props.poemId, load);
onMounted(load);

async function handleSubmit() {
  if (!canSubmit.value) return;
  saving.value = true;
  errorMsg.value = "";
  try {
    await new Promise((resolve) => setTimeout(resolve, 400));
    saved.value = { title: title.value, body: body.value };
    emit("save", { title: title.value.trim(), body: body.value });
  } catch {
    errorMsg.value = "保存失败，请重试";
  } finally {
    saving.value = false;
  }
}
</script>

<template>
  <form class="poem-editor" @submit.prevent="handleSubmit">
    <h2>{{ poemId ? "编辑诗歌" : "新建诗歌" }}</h2>

    <label class="field">
      <span>标题</span>
      <input v-model="title" :disabled="readonly || loading" placeholder="请输入诗题" />
    </label>

    <label class="field">
      <span>正文（每行一句）</span>
      <textarea
        v-model="body"
        rows="6"
        :disabled="readonly || loading"
        placeholder="请输入诗歌正文，每行一句"
      />
    </label>

    <p class="meta">
      {{ lines.length }} / {{ maxLines }} 行 · {{ charCount }} 字
      <span v-if="overLimit" class="error">超出行数限制，请删减诗句</span>
      <span v-else-if="dirty" class="hint">· 有未保存修改</span>
    </p>

    <p v-if="loading" class="hint">加载中…</p>
    <p v-else-if="errorMsg" class="error">{{ errorMsg }}</p>

    <div class="actions">
      <button type="submit" :disabled="!canSubmit">
        {{ saving ? "保存中…" : "保存" }}
      </button>
      <button type="button" :disabled="saving" @click="emit('cancel')">
        取消
      </button>
    </div>
  </form>
</template>

<style scoped>
.poem-editor {
  display: flex;
  flex-direction: column;
  gap: 12px;
  max-width: 32rem;
}
.field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 14px;
}
.field input,
.field textarea {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font: inherit;
}
.meta {
  font-size: 12px;
  color: #666;
}
.hint {
  color: #888;
}
.error {
  color: #c00;
}
.actions {
  display: flex;
  gap: 8px;
}
.actions button {
  padding: 6px 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: #fff;
  cursor: pointer;
}
.actions button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
