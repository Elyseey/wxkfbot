<template>
  <div class="panel">
    <el-card shadow="never" class="panel-card">
      <template #header>
        <div class="card-header">
          <span class="card-title">AI 模型配置</span>
          <el-button type="primary" @click="saveAIConfig" :loading="savingConfig">
            <el-icon><Check /></el-icon> 保存
          </el-button>
        </div>
      </template>
      <el-form label-width="100px" label-position="left">
        <el-form-item label="API Key">
          <el-input v-model="aiForm.apiKey" type="password" show-password placeholder="不修改请留空" />
          <span v-if="hasCustomApiKey" class="form-hint-inline">已配置自定义 Key</span>
        </el-form-item>
        <el-form-item label="模型名称">
          <el-autocomplete
            v-model="aiForm.model"
            :fetch-suggestions="queryModels"
            placeholder="输入模型名称"
            clearable
            style="width: 100%"
          >
            <template #append>
              <el-button :loading="loadingModels" @click="fetchModels" class="model-fetch-btn">拉取模型</el-button>
            </template>
          </el-autocomplete>
          <div v-if="modelList.length" class="model-count">
            <el-text type="info" size="small">{{ modelList.length }} 个可用模型</el-text>
          </div>
        </el-form-item>
        <el-form-item label="Base URL">
          <el-input v-model="aiForm.baseUrl" placeholder="如 https://api.openai.com/v1" />
        </el-form-item>
        <el-form-item label="超时时间">
          <el-input-number v-model="aiForm.timeout" :min="5000" :max="120000" :step="5000" />
          <span class="form-unit">ms</span>
        </el-form-item>
      </el-form>
      <div class="form-tip">修改后立即对新消息生效。留空的字段不会覆盖现有配置。</div>
    </el-card>

    <el-card shadow="never" class="panel-card" style="margin-top:16px">
      <template #header>
        <div class="card-header">
          <span class="card-title">系统提示词</span>
          <el-button type="primary" @click="savePrompt" :loading="saving">
            <el-icon><Check /></el-icon> 保存
          </el-button>
        </div>
      </template>
      <el-input
        v-model="systemPrompt"
        type="textarea"
        :rows="8"
        placeholder="输入系统提示词..."
      />
      <div class="form-tip">AI 回复时使用的系统提示词，修改后对新会话生效</div>
    </el-card>

    <el-card shadow="never" class="panel-card" style="margin-top:16px">
      <template #header>
        <div class="card-header">
          <span class="card-title">关键词 Webhook</span>
          <el-button type="primary" @click="addKeyword">
            <el-icon><Plus /></el-icon> 添加规则
          </el-button>
        </div>
      </template>
      <div class="webhook-rules" v-loading="loadingWebhooks">
        <div v-for="(rule, idx) in webhookRules" :key="idx" class="webhook-rule-card">
          <div class="rule-header">
            <el-switch v-model="rule.enabled" size="small" @change="markDirty(idx)" />
            <span class="rule-num">规则 {{ idx + 1 }}</span>
            <el-button link type="danger" size="small" @click="removeKeyword(idx)">删除</el-button>
          </div>
          <el-row :gutter="12">
            <el-col :span="8">
              <div class="rule-label">关键词</div>
              <el-input v-model="rule.keyword" size="small" placeholder="触发关键词" @input="markDirty(idx)" />
            </el-col>
            <el-col :span="6">
              <div class="rule-label">匹配方式</div>
              <el-select v-model="rule.match_mode" size="small" @change="markDirty(idx)" style="width:100%">
                <el-option value="exact" label="精确匹配" />
                <el-option value="contains" label="包含" />
                <el-option value="regex" label="正则" />
              </el-select>
            </el-col>
            <el-col :span="4">
              <div class="rule-label">方法</div>
              <el-select v-model="rule.method" size="small" @change="markDirty(idx)" style="width:100%">
                <el-option value="POST" label="POST" />
                <el-option value="GET" label="GET" />
              </el-select>
            </el-col>
            <el-col :span="6">
              <div class="rule-label">Content-Type</div>
              <el-select v-model="rule.content_type" size="small" @change="markDirty(idx)" style="width:100%">
                <el-option value="application/json" label="JSON" />
                <el-option value="application/x-www-form-urlencoded" label="Form" />
              </el-select>
            </el-col>
          </el-row>
          <div class="rule-label" style="margin-top:8px">Webhook URL</div>
          <el-input v-model="rule.webhook_url" size="small" placeholder="https://example.com/hook?user={{external_userid}}" @input="markDirty(idx)" />
          <div class="rule-label" style="margin-top:8px">请求体模板 <span class="rule-hint">（留空使用默认，支持变量）</span></div>
          <el-input v-model="rule.body_template" type="textarea" :rows="3" size="small"
            placeholder='{"text": "{{content}}", "user": "{{external_userid}}", "kfid": "{{open_kfid}}"}'
            @input="markDirty(idx)" />
        </div>
        <el-empty v-if="!webhookRules.length" :image-size="40" description="暂无规则" />
      </div>
      <div class="form-actions" v-if="webhookDirty" style="margin-top:12px">
        <el-button type="primary" @click="saveWebhooks" :loading="savingWebhooks">
          保存规则
        </el-button>
      </div>
      <div class="form-tip">
        支持变量: <code>{{content}}</code> 消息内容、<code>{{external_userid}}</code> 用户ID、<code>{{open_kfid}}</code> 客服ID、<code>{{msgid}}</code> 消息ID、<code>{{keyword}}</code> 匹配的关键词、<code>{{timestamp}}</code> 时间戳。GET 请求时变量可用于 URL 参数。
      </div>
    </el-card>

    <el-card shadow="never" class="panel-card" style="margin-top:16px">
      <template #header>
        <div class="card-header">
          <span class="card-title">关于</span>
        </div>
      </template>
      <div class="about-section">
        <p>微信客服机器人管理后台</p>
        <p class="about-link">
          <svg class="inline-icon" viewBox="0 0 16 16" width="14" height="14" fill="currentColor"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
          <a href="https://github.com/bestK/wxkfbot" target="_blank" rel="noopener">https://github.com/bestK/wxkfbot</a>
        </p>
      </div>
    </el-card>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { Refresh } from '@element-plus/icons-vue'
import { api } from '../api'

const saving = ref(false)
const savingConfig = ref(false)
const savingWebhooks = ref(false)
const loadingWebhooks = ref(false)
const systemPrompt = ref('')
const webhookDirty = ref(false)
const hasCustomApiKey = ref(false)

const aiForm = ref({ model: '', baseUrl: '', timeout: 30000, apiKey: '' })
const webhookRules = ref<any[]>([])
const modelList = ref<string[]>([])
const loadingModels = ref(false)

async function fetchModels() {
  loadingModels.value = true
  try {
    const res = await api.get('/ai_models')
    modelList.value = res?.models || []
    if (!modelList.value.length) ElMessage.info('未获取到模型列表')
  } catch (e: any) {
    ElMessage.error(e.message || '拉取模型失败')
  } finally {
    loadingModels.value = false
  }
}

function queryModels(query: string, cb: (results: { value: string }[]) => void) {
  const results = query
    ? modelList.value.filter(m => m.toLowerCase().includes(query.toLowerCase())).map(m => ({ value: m }))
    : modelList.value.map(m => ({ value: m }))
  cb(results)
}

async function loadStatus() {
  try {
    const res = await api.get('/ai_status')
    if (res) {
      aiForm.value.model = res.model || ''
      aiForm.value.baseUrl = res.baseUrl || ''
      aiForm.value.timeout = res.timeout || 30000
      aiForm.value.apiKey = ''
      hasCustomApiKey.value = !!res.hasCustomApiKey
      if (res.systemPrompt) systemPrompt.value = res.systemPrompt
    }
  } catch (e: any) {
    ElMessage.error(e.message)
  }
}

async function saveAIConfig() {
  savingConfig.value = true
  try {
    const payload: any = {}
    if (aiForm.value.model) payload.model = aiForm.value.model
    if (aiForm.value.baseUrl) payload.baseUrl = aiForm.value.baseUrl
    if (aiForm.value.timeout) payload.timeout = aiForm.value.timeout
    if (aiForm.value.apiKey) payload.apiKey = aiForm.value.apiKey
    await api.post('/ai_config', payload)
    ElMessage.success('AI 配置已更新')
    if (aiForm.value.apiKey) {
      hasCustomApiKey.value = true
      aiForm.value.apiKey = ''
    }
  } catch (e: any) {
    ElMessage.error(e.message)
  } finally {
    savingConfig.value = false
  }
}

async function savePrompt() {
  saving.value = true
  try {
    await api.post('/system_prompt', { prompt: systemPrompt.value })
    ElMessage.success('系统提示词已更新')
  } catch (e: any) {
    ElMessage.error(e.message)
  } finally {
    saving.value = false
  }
}

async function loadWebhooks() {
  loadingWebhooks.value = true
  try {
    const res = await api.get('/webhook_rules')
    webhookRules.value = res?.rules || []
  } catch (_) {
    webhookRules.value = []
  } finally {
    loadingWebhooks.value = false
  }
}

function addKeyword() {
  webhookRules.value.push({ keyword: '', match_mode: 'contains', webhook_url: '', enabled: true })
  webhookDirty.value = true
}

function removeKeyword(idx: number) {
  webhookRules.value.splice(idx, 1)
  webhookDirty.value = true
}

function markDirty(_idx: number) {
  webhookDirty.value = true
}

async function saveWebhooks() {
  const valid = webhookRules.value.every(r => r.keyword && r.webhook_url)
  if (!valid) {
    ElMessage.warning('请填写完整的关键词和 Webhook URL')
    return
  }
  savingWebhooks.value = true
  try {
    await api.post('/webhook_rules', { rules: webhookRules.value })
    ElMessage.success('Webhook 规则已保存')
    webhookDirty.value = false
  } catch (e: any) {
    ElMessage.error(e.message)
  } finally {
    savingWebhooks.value = false
  }
}

onMounted(() => {
  loadStatus()
  loadWebhooks()
})
</script>

<style scoped>
.panel-card { border-radius: var(--card-radius, 8px); }
.card-header { display: flex; align-items: center; justify-content: space-between; }
.card-title { font-weight: 600; font-size: 14px; color: var(--text-primary); }
.mono { font-family: var(--font-mono, 'JetBrains Mono', monospace); font-size: 13px; }
.form-tip { font-size: 12px; color: var(--text-muted, #999); margin-top: 8px; }
.form-tip code { font-family: var(--font-mono, 'JetBrains Mono', monospace); font-size: 11px; background: var(--bg-elevated, #f7f7f7); padding: 1px 4px; border-radius: 3px; }
.form-unit { margin-left: 8px; font-size: 13px; color: var(--text-secondary, #666); }
.form-actions { display: flex; justify-content: flex-end; }
.form-hint-inline { margin-left: 8px; font-size: 12px; color: var(--accent, #07C160); }

.model-picker { display: flex; align-items: center; gap: 8px; width: 100%; }
.model-select { flex: 1; }
.model-fetch-btn { color: var(--accent, #07C160) !important; background: #fff !important; border: 1px solid var(--border, #e0e0e0) !important; border-left: none !important; }
.model-count { margin-top: 4px; }

.webhook-rule-card { padding: 12px; margin-bottom: 12px; border: 1px solid var(--border, #e0e0e0); border-radius: var(--radius, 6px); background: var(--bg-elevated, #f7f7f7); }
.rule-header { display: flex; align-items: center; gap: 8px; margin-bottom: 10px; }
.rule-num { font-size: 13px; font-weight: 500; color: var(--text-secondary, #666); }
.rule-label { font-size: 12px; color: var(--text-muted, #999); margin-bottom: 4px; }
.rule-hint { font-size: 11px; color: var(--text-muted, #999); font-weight: normal; }

.about-section { font-size: 13px; color: var(--text-secondary, #666); }
.about-section p { margin-bottom: 6px; }
.about-link { display: flex; align-items: center; gap: 6px; }
.about-link a { color: var(--accent, #07C160); text-decoration: none; }
.about-link a:hover { text-decoration: underline; }
.inline-icon { vertical-align: middle; }
</style>
