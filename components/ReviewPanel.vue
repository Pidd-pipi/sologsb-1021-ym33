<script setup lang="ts">
import { computed, ref } from 'vue';
import { useDictionaryStore } from '~/store/dictionary';
import { fieldLabels, mergeStrategyLabels } from '~/utils/dictionary';

const store = useDictionaryStore();
const emit = defineEmits<{ versions: [] }>();
const commentField = ref('definition');
const commentText = ref('');
const filter = ref<'all' | 'open' | 'resolved'>('all');
const entry = computed(() => store.selectedEntry);
const comments = computed(() => (entry.value?.reviewerComments ?? []).filter((comment) => filter.value === 'all' || comment.status === filter.value));

const addComment = () => {
  if (!entry.value || !commentText.value.trim()) return;
  store.addComment(entry.value.id, commentField.value, commentText.value);
  commentText.value = '';
};
</script>

<template>
  <aside v-if="entry" class="panel review-panel">
    <div class="panel-head review-head">
      <div><span class="eyebrow">03 / REVIEW</span><h2>审校与回复</h2></div>
      <button class="version-link" @click="emit('versions')">版本 {{ store.versions.length }}</button>
    </div>
    <div class="review-summary">
      <div><strong>{{ entry.reviewerComments.filter((item) => item.status === 'open').length }}</strong><span>待处理</span></div>
      <div><strong>{{ entry.reviewerComments.filter((item) => item.status === 'resolved').length }}</strong><span>已解决</span></div>
      <div><strong>{{ entry.dialectVariants.length }}</strong><span>方言变体</span></div>
    </div>
    <div class="comment-filter">
      <button :class="{ active: filter === 'all' }" @click="filter = 'all'">全部</button>
      <button :class="{ active: filter === 'open' }" @click="filter = 'open'">待回复</button>
      <button :class="{ active: filter === 'resolved' }" @click="filter = 'resolved'">已解决</button>
    </div>
    <div v-if="entry.mergeHistory?.length" class="merge-history">
      <div class="merge-history-head"><strong>合并记录</strong><span>{{ entry.mergeHistory.length }} 次合并 · 字段取舍与意见来源</span></div>
      <article v-for="record in entry.mergeHistory" :key="record.id" class="merge-record">
        <header><time>{{ new Date(record.at).toLocaleString('zh-CN') }}</time><span>并入：{{ record.sourceHeadwords.join('、') }}</span></header>
        <ul>
          <li v-for="decision in record.fieldDecisions" :key="decision.field">
            <span class="decision-field">{{ fieldLabels[decision.field] || decision.field }}</span>
            <em class="decision-strategy" :data-strategy="decision.strategy">{{ mergeStrategyLabels[decision.strategy] }}</em>
            <small v-if="decision.strategy !== 'target'">主条「{{ decision.targetValue || '空' }}」/ 另一条「{{ decision.sourceValue || '空' }}」→「{{ decision.result || '空' }}」</small>
          </li>
        </ul>
      </article>
    </div>
    <div class="comment-list">
      <article v-for="comment in comments" :key="comment.id" class="comment-card" :class="{ resolved: comment.status === 'resolved' }">
        <header><t-tag size="small" variant="light" :theme="comment.status === 'open' ? 'warning' : 'success'">{{ fieldLabels[comment.field] || comment.field }}</t-tag><span>{{ comment.author }}</span><time>{{ new Date(comment.createdAt).toLocaleDateString('zh-CN') }}</time></header>
        <div v-if="comment.origin" class="comment-origin">来自原词条《{{ comment.origin.headword }}》 · 原字段「{{ fieldLabels[comment.origin.field] || comment.origin.field }}」</div>
        <p>{{ comment.message }}</p>
        <div v-for="reply in comment.replies" :key="reply.id" class="reply"><strong>{{ reply.author }}</strong><span>{{ reply.message }}</span><time>{{ new Date(reply.createdAt).toLocaleString('zh-CN') }}</time></div>
        <div class="reply-box">
          <t-textarea v-model="store.fieldReplyDrafts[comment.id]" :autosize="{ minRows: 1, maxRows: 3 }" placeholder="逐字段回复这条意见…" />
          <t-button size="small" theme="primary" variant="outline" @click="store.replyComment(entry!.id, comment.id, store.fieldReplyDrafts[comment.id] || ''); store.fieldReplyDrafts[comment.id] = ''">回复</t-button>
        </div>
        <button class="resolve-button" @click="store.toggleComment(entry!.id, comment.id)">{{ comment.status === 'open' ? '✓ 标记为解决' : '↺ 重新打开' }}</button>
      </article>
      <t-empty v-if="!comments.length" description="当前筛选下没有审校意见" />
    </div>
    <div class="new-comment">
      <div class="new-comment-title"><strong>新增逐字段意见</strong><span>Ctrl + Enter 提交</span></div>
      <t-select v-model="commentField" size="small">
        <t-option v-for="(label, field) in fieldLabels" :key="field" :value="field" :label="label" />
      </t-select>
      <t-textarea v-model="commentText" :autosize="{ minRows: 2, maxRows: 4 }" placeholder="指出需要修改、补充或确认的内容" @keydown.ctrl.enter="addComment" @keydown.meta.enter="addComment" />
      <t-button block theme="primary" size="small" :disabled="!commentText.trim()" @click="addComment">提交审校意见</t-button>
    </div>
  </aside>
</template>
