<script setup lang="ts">
import { computed, reactive, ref, watch } from 'vue';
import type { DuplicatePair, DictionaryEntry, MergeFieldChoice, ReviewComment } from '~/types/dictionary';
import { useDictionaryStore } from '~/store/dictionary';

const visible = defineModel<boolean>({ required: true });
const props = defineProps<{ pairs: DuplicatePair[] }>();
const store = useDictionaryStore();
const pairIndex = ref(0);
const targetSide = ref<'left' | 'right'>('left');
const choices = reactive<Record<string, MergeFieldChoice>>({
  headword: 'target', pronunciation: 'target', partOfSpeech: 'target', definition: 'target', notes: 'target'
});
const fields = [
  ['headword', '词形'], ['pronunciation', '发音'], ['partOfSpeech', '词性'], ['definition', '释义'], ['notes', '编者备注']
] as const;
const fieldLabels: Record<string, string> = {
  headword: '词形', pronunciation: '发音', partOfSpeech: '词性', definition: '释义', dialectVariants: '方言变体', examples: '例句', sources: '来源', synonyms: '同义词', notes: '编者备注'
};
const currentPair = computed(() => props.pairs[pairIndex.value]);
const left = computed<DictionaryEntry | undefined>(() => store.entries.find((entry) => entry.id === currentPair.value?.leftId));
const right = computed<DictionaryEntry | undefined>(() => store.entries.find((entry) => entry.id === currentPair.value?.rightId));
const target = computed(() => targetSide.value === 'left' ? left.value : right.value);
const source = computed(() => targetSide.value === 'left' ? right.value : left.value);
const openCommentsOf = (entry?: DictionaryEntry): ReviewComment[] => (entry?.reviewerComments ?? []).filter((comment) => comment.status === 'open');
const sides = computed(() => (left.value && right.value
  ? [{ name: '左侧', entry: left.value }, { name: '右侧', entry: right.value }]
  : []));

watch(visible, (open) => {
  if (!open) return;
  pairIndex.value = 0;
  targetSide.value = 'left';
  fields.forEach(([field]) => { choices[field] = 'target'; });
});

const confirmMerge = () => {
  if (!target.value || !source.value) return;
  store.mergeEntries(target.value.id, [source.value.id], choices);
  visible.value = false;
};
</script>

<template>
  <t-dialog v-model:visible="visible" header="重复词条并排比较" width="1120px" :footer="false" class="merge-dialog">
    <div v-if="currentPair && left && right" class="merge-content">
      <div class="merge-toolbar">
        <div class="pair-navigation"><span>疑似重复</span><strong>{{ pairIndex + 1 }} / {{ pairs.length }}</strong><t-button size="small" variant="outline" :disabled="pairIndex === 0" @click="pairIndex--">上一组</t-button><t-button size="small" variant="outline" :disabled="pairIndex >= pairs.length - 1" @click="pairIndex++">下一组</t-button></div>
        <div class="score-pill">{{ Math.round(currentPair.score * 100) }}% 相似</div>
        <span>{{ currentPair.reasons.join(' · ') }}</span>
      </div>

      <div class="merge-head">
        <div class="target-picker"><label><input v-model="targetSide" type="radio" value="left" /> 以左侧为主条</label></div>
        <div class="target-picker"><label><input v-model="targetSide" type="radio" value="right" /> 以右侧为主条</label></div>
      </div>

      <div class="compare-table">
        <div class="compare-row header"><span>字段</span><span>左侧 · {{ left.headword }}</span><span>右侧 · {{ right.headword }}</span><span>合并方式</span></div>
        <div v-for="[field, label] in fields" :key="field" class="compare-row" :class="{ conflict: left[field] !== right[field] }">
          <div class="field-name"><strong>{{ label }}</strong><small>{{ left[field] === right[field] ? '内容一致' : '字段冲突' }}</small></div>
          <div class="compare-value">{{ left[field] || '—' }}</div>
          <div class="compare-value">{{ right[field] || '—' }}</div>
          <t-radio-group v-model="choices[field]" variant="default" size="small">
            <t-radio-button value="target">主条</t-radio-button>
            <t-radio-button value="source">另一条</t-radio-button>
            <t-radio-button value="combine">拼接</t-radio-button>
          </t-radio-group>
        </div>
      </div>

      <div class="merge-layers">
        <div><strong>方言变体</strong><span>{{ left.dialectVariants.length }} + {{ right.dialectVariants.length }}</span><small>合并时全部保留</small></div>
        <div><strong>例句</strong><span>{{ left.examples.length }} + {{ right.examples.length }}</span><small>合并时全部保留</small></div>
        <div><strong>来源</strong><span>{{ left.sources.length }} + {{ right.sources.length }}</span><small>合并时全部保留</small></div>
        <div><strong>审校意见</strong><span>{{ left.reviewerComments.length }} + {{ right.reviewerComments.length }}</span><small>合并时全部保留并标注原词条、原字段</small></div>
      </div>

      <div class="merge-comments">
        <div class="merge-comments-head"><strong>未解决审校意见核对</strong><span>确认前请先核对两边待处理意见；合并后每条意见仍可看出原词条与原字段</span></div>
        <div class="merge-comment-cols">
          <div v-for="side in sides" :key="side.name" class="merge-comment-col">
            <div class="merge-comment-title">
              <strong>{{ side.name }} · {{ side.entry.headword }}</strong>
              <span>{{ openCommentsOf(side.entry).length }} 条未解决 · 共 {{ side.entry.reviewerComments.length }} 条</span>
            </div>
            <div v-for="comment in openCommentsOf(side.entry)" :key="comment.id" class="merge-comment-item">
              <t-tag size="small" variant="light" theme="warning">{{ fieldLabels[comment.field] || comment.field }}</t-tag><span>{{ comment.message }}</span>
            </div>
            <div v-if="!openCommentsOf(side.entry).length" class="merge-comment-empty">无未解决意见</div>
          </div>
        </div>
      </div>

      <div class="merge-warning"><strong>合并结果会标记为“争议”</strong><span>被合并词条不再单独显示；完整快照进入版本记录，字段取舍（主条 / 另一条 / 拼接）和每条意见的原词条、原字段随合并词条保存，可撤销或恢复。</span></div>
      <div class="dialog-actions"><t-button variant="outline" @click="visible = false">取消</t-button><t-button theme="primary" @click="confirmMerge">生成合并词条</t-button></div>
    </div>
    <t-empty v-else description="没有可合并的重复词条" />
  </t-dialog>
</template>
