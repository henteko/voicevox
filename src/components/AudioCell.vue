<template>
  <div class="audio-cell">
    <q-icon
      v-if="isActiveAudioCell"
      name="arrow_right"
      color="primary-light"
      size="sm"
      class="absolute active-arrow"
    />
    <div
      v-if="showTextLineNumber"
      class="line-number"
      :class="{ active: isActiveAudioCell }"
    >
      {{ textLineNumberIndex }}
    </div>
    <character-button
      v-model:selected-voice="selectedVoice"
      :character-infos="userOrderedCharacterInfos"
      :loading="isInitializingSpeaker"
      :show-engine-info="isMultipleEngine"
      :ui-locked="uiLocked"
      @focus="setActiveAudioKey()"
    />
    <!-- 
      input.valueをスクリプトから変更した場合は@changeが発火しないため、
      @blurと@keydown.prevent.enter.exactに分けている
    -->
    <q-input
      ref="textfield"
      filled
      dense
      hide-bottom-space
      class="full-width"
      color="primary-light"
      :disable="uiLocked"
      :error="audioTextBuffer.length >= 80"
      :model-value="audioTextBuffer"
      :aria-label="`${textLineNumberIndex}行目`"
      @update:model-value="setAudioTextBuffer"
      @focus="
        clearInputSelection();
        setActiveAudioKey();
      "
      @blur="pushAudioTextIfNeeded()"
      @paste="pasteOnAudioCell"
      @keydown.prevent.up.exact="moveUpCell"
      @keydown.prevent.down.exact="moveDownCell"
      @keydown.prevent.enter.exact="pushAudioTextIfNeeded"
    >
      <template #error>
        文章が長いと正常に動作しない可能性があります。
        句読点の位置で文章を分割してください。
      </template>
      <template v-if="deleteButtonEnable" #after>
        <q-btn
          round
          flat
          icon="delete_outline"
          size="0.8rem"
          :disable="uiLocked"
          :aria-label="`${textLineNumberIndex}行目を削除`"
          @click="removeCell"
        />
      </template>
      <context-menu
        ref="contextMenu"
        :header="contextMenuHeader"
        :menudata="contextMenudata"
        @before-show="
          startContextMenuOperation();
          readyForContextMenu();
        "
        @before-hide="endContextMenuOperation()"
      />
    </q-input>
  </div>
</template>

<script setup lang="ts">
import { computed, watch, ref, nextTick } from "vue";
import { QInput } from "quasar";
import CharacterButton from "./CharacterButton.vue";
import { MenuItemButton, MenuItemSeparator } from "./MenuBar.vue";
import ContextMenu from "./ContextMenu.vue";
import { useStore } from "@/store";
import { AudioKey, SplitTextWhenPasteType, Voice } from "@/type/preload";
import { SelectionHelperForQInput } from "@/helpers/SelectionHelperForQInput";

const props =
  defineProps<{
    audioKey: AudioKey;
  }>();

const emit =
  defineEmits<{
    (e: "focusCell", payload: { audioKey: AudioKey }): void;
  }>();

defineExpose({
  audioKey: computed(() => props.audioKey),
  focusTextField: () => {
    textfield.value?.focus();
  },
  removeCell: () => {
    removeCell();
  },
});

const store = useStore();
const userOrderedCharacterInfos = computed(() => {
  const infos = store.getters.USER_ORDERED_CHARACTER_INFOS;
  if (infos == undefined)
    throw new Error("USER_ORDERED_CHARACTER_INFOS == undefined");
  return infos;
});
const isInitializingSpeaker = computed(
  () => store.state.audioKeyInitializingSpeaker === props.audioKey
);
const audioItem = computed(() => store.state.audioItems[props.audioKey]);

const uiLocked = computed(() => store.getters.UI_LOCKED);

const selectedVoice = computed<Voice | undefined>({
  get() {
    const { engineId, styleId } = audioItem.value.voice;

    if (
      !store.state.engineIds.some((storeEngineId) => storeEngineId === engineId)
    )
      return undefined;

    const speakerInfo =
      userOrderedCharacterInfos.value != undefined
        ? store.getters.CHARACTER_INFO(engineId, styleId)
        : undefined;

    if (speakerInfo == undefined) return undefined;
    return { engineId, speakerId: speakerInfo.metas.speakerUuid, styleId };
  },
  set(voice: Voice | undefined) {
    if (voice == undefined) return;
    store.dispatch("COMMAND_CHANGE_VOICE", {
      audioKey: props.audioKey,
      voice,
    });
  },
});

const isActiveAudioCell = computed(
  () => props.audioKey === store.getters.ACTIVE_AUDIO_KEY
);

const audioTextBuffer = ref(audioItem.value.text);
const isChangeFlag = ref(false);
const setAudioTextBuffer = (text: string | number | null) => {
  if (typeof text !== "string") throw new Error("typeof text !== 'string'");
  audioTextBuffer.value = text;
  isChangeFlag.value = true;
};

watch(
  // `audioItem` becomes undefined just before the component is unmounted.
  () => audioItem.value?.text,
  (newText) => {
    if (!isChangeFlag.value && newText !== undefined) {
      audioTextBuffer.value = newText;
    }
  }
);

const pushAudioTextIfNeeded = async (event?: KeyboardEvent) => {
  if (event && event.isComposing) return;
  if (!willRemove.value && isChangeFlag.value && !willFocusOrBlur.value) {
    isChangeFlag.value = false;
    await store.dispatch("COMMAND_CHANGE_AUDIO_TEXT", {
      audioKey: props.audioKey,
      text: audioTextBuffer.value,
    });
  }
};

// バグ修正用
// see https://github.com/VOICEVOX/voicevox/pull/1364#issuecomment-1620594931
const clearInputSelection = () => {
  if (!willFocusOrBlur.value) {
    textfieldSelection.toEmpty();
  }
};

const setActiveAudioKey = () => {
  store.dispatch("SET_ACTIVE_AUDIO_KEY", { audioKey: props.audioKey });
};

// コピペしたときに句点と改行で区切る
const textSplitType = computed(() => store.state.splitTextWhenPaste);
const pasteOnAudioCell = async (event: ClipboardEvent) => {
  event.preventDefault();
  paste({ text: event.clipboardData?.getData("text/plain") });
};
/**
 * 貼り付け。
 * ブラウザ版を考えるとClipboard APIをなるべく回避したいため、積極的に`options.text`を指定してください。
 */
const paste = async (options?: { text?: string }) => {
  const text = options ? options.text : await navigator.clipboard.readText();
  if (text === undefined) return;

  // 複数行貼り付けできるか試す
  if (textSplitType.value !== "OFF") {
    const textSplitter: Record<
      SplitTextWhenPasteType,
      (text: string) => string[]
    > = {
      PERIOD_AND_NEW_LINE: (text) =>
        text.replaceAll("。", "。\r\n").split(/[\r\n]/),
      NEW_LINE: (text) => text.split(/[\r\n]/),
      OFF: (text) => [text],
    };
    const texts = textSplitter[textSplitType.value](text);

    if (texts.length >= 2 && texts.some((text) => text !== "")) {
      await putMultilineText(texts);
      return;
    }
  }

  const beforeLength = audioTextBuffer.value.length;
  const end = textfieldSelection.selectionEnd ?? 0;
  setAudioTextBuffer(textfieldSelection.getReplacedStringTo(text));
  await nextTick();
  // 自動的に削除される改行などの文字数を念のため考慮している
  textfieldSelection.setCursorPosition(
    end + audioTextBuffer.value.length - beforeLength
  );
};
const putMultilineText = async (texts: string[]) => {
  // フォーカスを外して編集中のテキスト内容を確定させる
  if (document.activeElement instanceof HTMLInputElement) {
    document.activeElement.blur();
  }

  const prevAudioKey = props.audioKey;
  if (audioTextBuffer.value == "") {
    const text = texts.shift();
    if (text == undefined) throw new Error("予期せぬタイプエラーです。");
    setAudioTextBuffer(text);
    await pushAudioTextIfNeeded();
  }

  const audioKeys = await store.dispatch("COMMAND_PUT_TEXTS", {
    texts,
    voice: audioItem.value.voice,
    prevAudioKey,
  });
  if (audioKeys.length > 0) {
    emit("focusCell", { audioKey: audioKeys[audioKeys.length - 1] });
  }
};

// 行番号を表示するかどうか
const showTextLineNumber = computed(() => store.state.showTextLineNumber);
// 行番号
const textLineNumberIndex = computed(
  () => audioKeys.value.indexOf(props.audioKey) + 1
);
// 行番号の幅: 2桁はデフォで入るように, 3桁以上は1remずつ広げる
const textLineNumberWidth = computed(() => {
  const indexDigits = String(audioKeys.value.length).length;
  if (indexDigits <= 2) return "1.5rem";
  return `${indexDigits - 0.5}rem`;
});

// 上下に移動
const audioKeys = computed(() => store.state.audioKeys);
const moveUpCell = (e?: KeyboardEvent) => {
  if (e && e.isComposing) return;
  const index = audioKeys.value.indexOf(props.audioKey) - 1;
  if (index >= 0) {
    emit("focusCell", { audioKey: audioKeys.value[index] });
  }
};
const moveDownCell = (e?: KeyboardEvent) => {
  if (e && e.isComposing) return;
  const index = audioKeys.value.indexOf(props.audioKey) + 1;
  if (index < audioKeys.value.length) {
    emit("focusCell", { audioKey: audioKeys.value[index] });
  }
};

// 消去
const willRemove = ref(false);
const removeCell = async () => {
  // 1つだけの時は削除せず
  if (audioKeys.value.length > 1) {
    // フォーカスを外したりREMOVEしたりすると、
    // テキストフィールドのchangeイベントが非同期に飛んでundefinedエラーになる
    // エラー防止のためにまずwillRemoveフラグを建てる
    willRemove.value = true;

    const index = audioKeys.value.indexOf(props.audioKey);
    if (index > 0) {
      emit("focusCell", { audioKey: audioKeys.value[index - 1] });
    } else {
      emit("focusCell", { audioKey: audioKeys.value[index + 1] });
    }

    store.dispatch("COMMAND_REMOVE_AUDIO_ITEM", {
      audioKey: props.audioKey,
    });
  }
};

// 削除ボタンの有効／無効判定
const deleteButtonEnable = computed(() => {
  return 1 < audioKeys.value.length;
});

// テキスト編集エリアの右クリック
const contextMenu = ref<InstanceType<typeof ContextMenu>>();

// FIXME: 可能なら`isRangeSelected`と`contextMenuHeader`をcomputedに
const isRangeSelected = ref(false);
const contextMenuHeader = ref<string | undefined>("");
const contextMenudata = ref<
  [
    MenuItemButton,
    MenuItemButton,
    MenuItemButton,
    MenuItemSeparator,
    MenuItemButton,
    MenuItemSeparator,
    MenuItemButton
  ]
>([
  // NOTE: audioTextBuffer.value の変更が nativeEl.value に反映されるのはnextTick。
  {
    type: "button",
    label: "切り取り",
    onClick: async () => {
      contextMenu.value?.hide();
      if (textfieldSelection.isEmpty) return;

      const text = textfieldSelection.getAsString();
      const start = textfieldSelection.selectionStart;
      setAudioTextBuffer(textfieldSelection.getReplacedStringTo(""));
      await nextTick();
      navigator.clipboard.writeText(text);
      textfieldSelection.setCursorPosition(start);
    },
    disableWhenUiLocked: true,
  },
  {
    type: "button",
    label: "コピー",
    onClick: () => {
      contextMenu.value?.hide();
      if (textfieldSelection.isEmpty) return;

      navigator.clipboard.writeText(textfieldSelection.getAsString());
    },
    disableWhenUiLocked: true,
  },
  {
    type: "button",
    label: "貼り付け",
    onClick: async () => {
      contextMenu.value?.hide();
      paste();
    },
    disableWhenUiLocked: true,
  },
  { type: "separator" },
  {
    type: "button",
    label: "全選択",
    onClick: async () => {
      contextMenu.value?.hide();
      textfield.value?.select();
    },
    disableWhenUiLocked: true,
  },
  { type: "separator" },
  {
    type: "button",
    label: "内容をテキストのみに適用",
    onClick: async () => {
      contextMenu.value?.hide();
      isChangeFlag.value = false;
      await store.dispatch("COMMAND_CHANGE_DISPLAY_TEXT", {
        audioKey: props.audioKey,
        text: audioTextBuffer.value,
      });
      textfield.value?.blur();
    },
    disableWhenUiLocked: true,
  },
]);
/**
 * コンテキストメニューの開閉によりFocusやBlurが発生する可能性のある間は`true`。
 */
// no-focus を付けた場合と付けてない場合でタイミングが異なるため、両方に対応。
const willFocusOrBlur = ref(false);
const startContextMenuOperation = () => {
  willFocusOrBlur.value = true;
};
const readyForContextMenu = () => {
  const getMenuItemButton = (label: string) => {
    const item = contextMenudata.value.find((item) => item.label === label);
    if (item?.type !== "button")
      throw new Error("コンテキストメニューアイテムの取得に失敗しました。");
    return item;
  };

  const MAX_HEADER_LENGTH = 15;
  const SHORTED_HEADER_FRAGMENT_LENGTH = 5;

  // 選択範囲を1行目に表示
  const selectionText = textfieldSelection.getAsString();
  if (selectionText.length === 0) {
    isRangeSelected.value = false;
    getMenuItemButton("切り取り").disabled = true;
    getMenuItemButton("コピー").disabled = true;
  } else {
    isRangeSelected.value = true;
    getMenuItemButton("切り取り").disabled = false;
    getMenuItemButton("コピー").disabled = false;
    if (selectionText.length > MAX_HEADER_LENGTH) {
      // 長すぎる場合適度な長さで省略
      contextMenuHeader.value =
        selectionText.length <= MAX_HEADER_LENGTH
          ? selectionText
          : `${selectionText.substring(
              0,
              SHORTED_HEADER_FRAGMENT_LENGTH
            )} ... ${selectionText.substring(
              selectionText.length - SHORTED_HEADER_FRAGMENT_LENGTH
            )}`;
    } else {
      contextMenuHeader.value = selectionText;
    }
  }
};
const endContextMenuOperation = async () => {
  await nextTick();
  willFocusOrBlur.value = false;
};

// テキスト欄
const textfield = ref<QInput>();
const textfieldSelection = new SelectionHelperForQInput(textfield);

// 複数エンジン
const isMultipleEngine = computed(() => store.state.engineIds.length > 1);
</script>

<style scoped lang="scss">
@use '@/styles/visually-hidden' as visually-hidden;
@use '@/styles/colors' as colors;

.audio-cell {
  display: flex;
  padding: 0.4rem 0.5rem;
  margin: 0.2rem 0.5rem;

  &:first-child {
    margin-top: 0.6rem;
  }

  &:last-child {
    margin-bottom: 0.6rem;
  }

  gap: 0px 1rem;

  .active-arrow {
    left: -5px;
    height: 2rem;
  }

  .line-number {
    height: 2rem;
    width: v-bind(textLineNumberWidth);
    line-height: 2rem;
    margin-right: -0.3rem;
    opacity: 0.6;
    text-align: right;
    color: colors.$display;
    &.active {
      opacity: 1;
      font-weight: bold;
      color: colors.$primary-light;
    }
  }

  .q-input {
    :deep(.q-field__control) {
      height: 2rem;
      background: none;
      border-bottom: 1px solid colors.$primary-light;

      &::before {
        border-bottom: none;
      }
    }

    :deep(.q-field__after) {
      height: 2rem;
      padding-left: 5px;
    }

    &.q-field--filled.q-field--highlighted :deep(.q-field__control):before {
      background-color: rgba(colors.$display-rgb, 0.08);
    }
  }

  &:not(:hover) > .q-input > .q-field__after > .q-btn:not(:focus):not(:active) {
    @include visually-hidden.visually-hidden;
  }

  :deep(input) {
    caret-color: colors.$display;
    color: colors.$display;
  }
}
</style>
