<script setup lang="ts">
import type { PropType } from "vue"
import { onMounted, ref, watch, watchEffect } from "vue"
import { Compartment, EditorState } from "@codemirror/state"
import { EditorView } from "@codemirror/view"
import type { LanguageId } from "@/core/languages"
import { loadCodemirrorLanguageSupport } from "@/core/languages"
import type { ParseError } from "@/core/types"
import { useIsDark } from "@/composables/useIsDark"
import {
  defaultExtensions,
  rangesHighlighting,
  updateListeners,
} from "./extensions"
import { HighlightEffect } from "./extensions/rangesHighlighting"
import { errorPanelState, ShowErrorEffect } from "./extensions/errorPanel"
import { ThemeManager } from "./extensions/themes/manager"
import { basicDark } from "./extensions/themes/dark"
import { basicLight } from "./extensions/themes/light"

const props = defineProps({
  modelValue: {
    type: String,
    required: false,
    default: "",
  },
  languageId: {
    type: String as PropType<LanguageId>,
    required: true,
  },
  autofocus: Boolean,
  parseError: {
    type: Object as PropType<ParseError>,
    required: false,
  },
})

const emit = defineEmits(["update:modelValue", "blur"])
const container = ref<HTMLDivElement | null>(null)
const editor = {
  state: null as EditorState | null,
  view: null as EditorView | null,
}
const language = new Compartment()
const isDark = useIsDark()

const handleDocChange = (newDoc: string) => {
  emit("update:modelValue", newDoc)
}

onMounted(() => {
  if (container.value == null) {
    throw new Error(
      "Failed to mount codemirror editor, container element is null"
    )
  }

  editor.state = EditorState.create({
    doc: props.modelValue,
    extensions: [
      defaultExtensions,
      language.of([]),
      rangesHighlighting,
      errorPanelState,
      updateListeners({
        onChange: handleDocChange,
        onBlur: () => {
          emit("blur")
        },
      }),
    ],
  })

  editor.view = new EditorView({
    state: editor.state,
    parent: container.value,
  })

  watchEffect(() => {
    editor.view?.dispatch({
      effects: ThemeManager.reconfigure(isDark.value ? basicDark : basicLight),
    })
  })

  // Handle language change
  watch(
    () => props.languageId,
    async (newLanguage) => {
      const languageSupport = await loadCodemirrorLanguageSupport(newLanguage)
      if (languageSupport) {
        editor.view?.dispatch({
          effects: language.reconfigure(languageSupport),
        })
      } else {
        // TODO: handle case when there is no codemirror language support
      }
    },
    { immediate: true }
  )

  watch(
    () => props.parseError,
    (newParseError) => {
      editor.view?.dispatch({
        effects: [
          HighlightEffect.of({
            type: "error",
            loc: newParseError?.location && {
              from: newParseError.location.start.index,
              to: newParseError.location.end.index,
            },
          }),
          ShowErrorEffect.of(newParseError ?? null),
        ],
      })
    },
    { immediate: true }
  )

  if (props.autofocus) {
    editor.view.focus()
  }
})
</script>

<template>
  <div ref="container" class="editor-container" />
</template>

<style lang="scss">
.codemirror-highlighted_error {
  text-decoration: underline;
  text-decoration-style: wavy;
  text-decoration-color: var(--color-danger);
  text-decoration-thickness: 0.1rem;
}

.codemirror-error-panel {
  font-size: 0.85rem;
  padding: 2px 16px;
  color: var(--color-danger);

  &__with-location {
    cursor: pointer;
  }
}

.cm-gutters {
  border-right: 1px solid var(--color-primary) !important;
  background: var(--color-secondary) !important;
}

.cm-activeLineGutter {
  background: var(--color-primary) !important;
}

.ͼo,
.ͼ1o,
.ͼ1n {
  background: none;
}
</style>
