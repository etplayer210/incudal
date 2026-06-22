<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import api from '@/api'
import SkeletonLoader from '@/components/SkeletonLoader.vue'
import { useToast } from '@/stores/toast'
import { useThemeStore } from '@/stores/theme'
import { useAuthStore } from '@/stores/auth'
import { translateError } from '@/utils/errorHandler'
import type { Package } from '@/types/api'

const props = defineProps<{
  pkg: Package
}>()

const { t } = useI18n()
const toast = useToast()
const themeStore = useThemeStore()
const authStore = useAuthStore()

const MAX_PACKAGE_PLAN_PRICE_CENTS = 99999999
const MAX_PACKAGE_PLAN_PRICE = MAX_PACKAGE_PLAN_PRICE_CENTS / 100

interface PackagePlan {
  id: number
  name: string
  description: string | null
  cpu: number
  memory: number
  disk: number
  portLimit: number
  snapshotLimit: number
  backupLimit: number
  siteLimit: number
  swapSize: number
  trafficLimit: string
  trafficLimitSpeed: string
  price: number
  billingCycle: number
  trafficResetEnabled: boolean
  trafficResetPrice: number
  monthlyPrice: number
  isActive: boolean
  isSoldOut: boolean
  sortOrder: number
  slaGuarantee: number | null
}

type PackagePlanStatus = 'active' | 'soldOut' | 'inactive'

const plans = ref<PackagePlan[]>([])
const plansLoading = ref(false)
const showPlanFormModal = ref(false)
const editingPlan = ref<PackagePlan | null>(null)
const planFormLoading = ref(false)

const planForm = ref({
  name: '',
  description: '',
  cpu: 100,
  memory: 512,
  disk: 5120,
  portLimit: 3,
  snapshotLimit: 3,
  swapSize: 0,
  siteLimit: 1,
  trafficLimit: 1,
  trafficLimitSpeed: 10,
  price: 0,
  billingCycle: 1,
  trafficResetEnabled: false,
  trafficResetPrice: 0,
  status: 'active' as PackagePlanStatus,
  sortOrder: 0,
  slaGuarantee: null as number | null
})

const planStatusOptions = computed(() => [
  { value: 'active' as PackagePlanStatus, label: t('resources.plans.statusActive'), description: t('resources.plans.statusActiveHint') },
  { value: 'soldOut' as PackagePlanStatus, label: t('resources.plans.statusSoldOut'), description: t('resources.plans.statusSoldOutHint') },
  { value: 'inactive' as PackagePlanStatus, label: t('resources.plans.statusInactive'), description: t('resources.plans.statusInactiveHint') }
])

const selectedPlanStatusOption = computed(() =>
  planStatusOptions.value.find(option => option.value === planForm.value.status) || planStatusOptions.value[0]
)

const billingCycleOptions = computed(() => {
  if (!authStore.isAdmin) {
    return [{ value: 1, label: t('resources.plans.monthly') }]
  }
  return [
    { value: 1, label: t('resources.plans.monthly') },
    { value: 3, label: t('resources.plans.quarterly') },
    { value: 6, label: t('resources.plans.semiAnnual') },
    { value: 12, label: t('resources.plans.yearly') }
  ]
})

onMounted(() => {
  void loadPlans()
})

watch(() => props.pkg.id, () => {
  showPlanFormModal.value = false
  editingPlan.value = null
  void loadPlans()
})

async function loadPlans(): Promise<void> {
  plansLoading.value = true
  try {
    const res = await api.packages.getPlans(props.pkg.id)
    plans.value = (res.plans || []) as PackagePlan[]
  } catch (err) {
    console.error('加载方案列表失败:', err)
    plans.value = []
  } finally {
    plansLoading.value = false
  }
}

function getPlanStatus(plan: PackagePlan): PackagePlanStatus {
  if (!plan.isActive) return 'inactive'
  if (plan.isSoldOut) return 'soldOut'
  return 'active'
}

function getPlanStatusLabel(plan: PackagePlan): string {
  const status = getPlanStatus(plan)
  if (status === 'soldOut') return t('resources.plans.statusSoldOut')
  if (status === 'inactive') return t('resources.plans.statusInactive')
  return t('resources.plans.statusActive')
}

function getPlanStatusBadgeClass(plan: PackagePlan): string {
  const status = getPlanStatus(plan)
  if (status === 'soldOut') {
    return themeStore.isDark ? 'bg-red-500/15 text-red-300' : 'bg-red-50 text-red-700'
  }
  if (status === 'inactive') {
    return themeStore.isDark ? 'bg-gray-700 text-gray-300' : 'bg-gray-100 text-gray-600'
  }
  return 'badge-success'
}

function getPlanStatusSegmentClass(status: PackagePlanStatus): string {
  const selected = planForm.value.status === status

  if (!selected) {
    return themeStore.isDark
      ? 'text-gray-400 hover:bg-gray-800 hover:text-gray-200'
      : 'text-gray-500 hover:bg-white hover:text-gray-900'
  }

  if (status === 'active') {
    return themeStore.isDark
      ? 'bg-emerald-500/15 text-emerald-300 shadow-sm ring-1 ring-emerald-500/30'
      : 'bg-white text-emerald-700 shadow-sm ring-1 ring-emerald-200'
  }

  if (status === 'soldOut') {
    return themeStore.isDark
      ? 'bg-amber-500/15 text-amber-300 shadow-sm ring-1 ring-amber-500/30'
      : 'bg-white text-amber-700 shadow-sm ring-1 ring-amber-200'
  }

  return themeStore.isDark
    ? 'bg-gray-700 text-gray-200 shadow-sm ring-1 ring-gray-600'
    : 'bg-white text-gray-700 shadow-sm ring-1 ring-gray-200'
}

function getPlanStatusDotClass(status: PackagePlanStatus): string {
  if (status === 'active') return themeStore.isDark ? 'bg-emerald-300' : 'bg-emerald-500'
  if (status === 'soldOut') return themeStore.isDark ? 'bg-amber-300' : 'bg-amber-500'
  return themeStore.isDark ? 'bg-gray-400' : 'bg-gray-500'
}

function normalizePlanPriceCents(value: unknown): number | null {
  const price = Number(value)
  if (!Number.isFinite(price) || price < 0 || price > MAX_PACKAGE_PLAN_PRICE) {
    return null
  }

  const cents = price * 100
  if (Math.abs(cents - Math.round(cents)) >= 1e-8) {
    return null
  }

  const roundedCents = Math.round(cents)
  return roundedCents <= MAX_PACKAGE_PLAN_PRICE_CENTS ? roundedCents : null
}

function openCreatePlanModal(): void {
  editingPlan.value = null
  planForm.value = {
    name: '',
    description: '',
    cpu: props.pkg.cpu_max || 100,
    memory: props.pkg.memory_max || 512,
    disk: props.pkg.disk_max || 5120,
    portLimit: 3,
    snapshotLimit: 3,
    swapSize: 0,
    siteLimit: 1,
    trafficLimit: 1,
    trafficLimitSpeed: 10,
    price: 0,
    billingCycle: 1,
    trafficResetEnabled: false,
    trafficResetPrice: 0,
    status: 'active',
    sortOrder: plans.value.length,
    slaGuarantee: null
  }
  showPlanFormModal.value = true
}

function openEditPlanModal(plan: PackagePlan): void {
  editingPlan.value = plan
  planForm.value = {
    name: plan.name,
    description: plan.description || '',
    cpu: plan.cpu,
    memory: plan.memory,
    disk: plan.disk,
    portLimit: plan.portLimit,
    snapshotLimit: plan.snapshotLimit,
    swapSize: plan.swapSize,
    siteLimit: plan.siteLimit,
    trafficLimit: bytesToGB(plan.trafficLimit),
    trafficLimitSpeed: bytesToMbps(plan.trafficLimitSpeed),
    price: plan.price / 100,
    billingCycle: plan.billingCycle,
    trafficResetEnabled: plan.trafficResetEnabled,
    trafficResetPrice: plan.trafficResetPrice / 100,
    status: getPlanStatus(plan),
    sortOrder: plan.sortOrder,
    slaGuarantee: plan.slaGuarantee
  }
  showPlanFormModal.value = true
}

function bytesToGB(bytes: string | null | undefined): number {
  if (!bytes) return 1
  try {
    const b = Number(bytes)
    if (isNaN(b) || b <= 0) return 1
    const gb = b / (1024 * 1024 * 1024)
    return Math.round(gb * 1000000) / 1000000 || 1
  } catch {
    return 1
  }
}

function bytesToMbps(bytes: string | null | undefined): number {
  if (!bytes || bytes === '0') return 10
  try {
    const b = Number(bytes)
    if (isNaN(b) || b <= 0) return 10
    return Math.round(b / (1024 * 1024)) || 10
  } catch {
    return 10
  }
}

function gbToBytes(gb: string | number): string {
  if (gb === '' || gb === null || gb === undefined) return ''
  try {
    const gbNum = typeof gb === 'string' ? parseFloat(gb) : gb
    if (isNaN(gbNum) || gbNum <= 0) return ''
    return Math.floor(gbNum * 1024 * 1024 * 1024).toString()
  } catch {
    return ''
  }
}

function mbpsToBytes(mbps: string | number): string {
  if (mbps === '' || mbps === null || mbps === undefined) return ''
  try {
    const mbpsNum = typeof mbps === 'string' ? parseFloat(mbps) : mbps
    if (isNaN(mbpsNum) || mbpsNum <= 0) return ''
    return Math.floor(mbpsNum * 1024 * 1024).toString()
  } catch {
    return ''
  }
}

async function savePlan(): Promise<void> {
  if (!planForm.value.name.trim()) return

  const priceCents = normalizePlanPriceCents(planForm.value.price)
  if (priceCents === null) {
    toast.error(t('resources.plans.priceRangeError', { max: MAX_PACKAGE_PLAN_PRICE.toFixed(2) }))
    return
  }
  planForm.value.price = priceCents / 100

  const trafficResetPriceCents = planForm.value.trafficResetEnabled
    ? normalizePlanPriceCents(planForm.value.trafficResetPrice)
    : 0
  if (trafficResetPriceCents === null) {
    toast.error(t('resources.plans.trafficResetPriceRangeError', { max: MAX_PACKAGE_PLAN_PRICE.toFixed(2) }))
    return
  }
  planForm.value.trafficResetPrice = trafficResetPriceCents / 100

  planFormLoading.value = true
  try {
    const data = {
      name: planForm.value.name.trim(),
      description: planForm.value.description.trim() || undefined,
      cpu: planForm.value.cpu,
      memory: planForm.value.memory,
      disk: planForm.value.disk,
      portLimit: planForm.value.portLimit,
      snapshotLimit: planForm.value.snapshotLimit,
      backupLimit: 0,
      siteLimit: planForm.value.siteLimit,
      swapSize: planForm.value.swapSize,
      trafficLimit: gbToBytes(planForm.value.trafficLimit) || '0',
      trafficLimitSpeed: mbpsToBytes(planForm.value.trafficLimitSpeed) || '0',
      price: priceCents,
      billingCycle: planForm.value.billingCycle,
      trafficResetEnabled: planForm.value.trafficResetEnabled,
      trafficResetPrice: trafficResetPriceCents,
      isActive: planForm.value.status !== 'inactive',
      isSoldOut: planForm.value.status === 'soldOut',
      sortOrder: planForm.value.sortOrder,
      slaGuarantee: planForm.value.slaGuarantee ?? undefined
    }

    if (editingPlan.value) {
      await api.packages.updatePlan(props.pkg.id, editingPlan.value.id, data)
      toast.success(t('resources.plans.updateSuccess'))
    } else {
      await api.packages.createPlan(props.pkg.id, data)
      toast.success(t('resources.plans.createSuccess'))
    }

    showPlanFormModal.value = false
    await loadPlans()
  } catch (err: any) {
    toast.error(translateError(err) || t('resources.plans.saveFailed'))
  } finally {
    planFormLoading.value = false
  }
}

async function deletePlan(plan: PackagePlan): Promise<void> {
  if (!confirm(t('resources.plans.confirmDelete', { name: plan.name }))) return

  try {
    await api.packages.deletePlan(props.pkg.id, plan.id)
    toast.success(t('resources.plans.deleteSuccess'))
    await loadPlans()
  } catch (err: any) {
    toast.error(err.message || t('resources.plans.deleteFailed'))
  }
}

function formatMemory(mb: number): string {
  if (!mb) return '0'
  if (mb >= 1024) {
    const gb = mb / 1024
    return gb % 1 === 0 ? `${gb.toFixed(0)} GB` : `${gb.toFixed(1)} GB`
  }
  return `${mb} MB`
}

function formatDisk(mb: number): string {
  if (!mb) return '0'
  if (mb >= 1024) {
    const gb = mb / 1024
    return gb % 1 === 0 ? `${gb.toFixed(0)} GB` : `${gb.toFixed(1)} GB`
  }
  return `${mb} MB`
}

function formatPrice(cents: number): string {
  return (cents / 100).toFixed(2)
}

function getBillingCycleLabel(months: number): string {
  const opt = billingCycleOptions.value.find(o => o.value === months)
  return opt ? opt.label : `${months} ${t('resources.plans.months')}`
}
</script>

<template>
  <div class="space-y-4">
    <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h2 class="text-lg font-medium text-themed">{{ t('resources.plans.title') }}</h2>
        <p class="text-sm text-themed-muted">{{ t('resources.plans.noPlansHint') }}</p>
      </div>
      <button type="button" class="btn-primary" @click="openCreatePlanModal">
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
        </svg>
        {{ t('resources.plans.add') }}
      </button>
    </div>

    <SkeletonLoader v-if="plansLoading" type="list" />

    <div v-else-if="plans.length === 0" class="card p-12 text-center">
      <div class="w-20 h-20 mx-auto mb-4 rounded-2xl flex items-center justify-center" :class="themeStore.isDark ? 'bg-gray-800' : 'bg-gray-100'">
        <svg class="w-10 h-10" :class="themeStore.isDark ? 'text-gray-600' : 'text-gray-400'" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2" />
        </svg>
      </div>
      <p class="text-themed-secondary font-medium">{{ t('resources.plans.noPlans') }}</p>
      <p class="text-xs text-themed-muted mt-1">{{ t('resources.plans.noPlansHint') }}</p>
    </div>

    <div v-else class="space-y-3">
      <div
        v-for="plan in plans"
        :key="plan.id"
        class="card p-4 transition-colors"
        :class="getPlanStatus(plan) !== 'active' ? 'opacity-70' : ''"
      >
        <div class="flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between">
          <div class="min-w-0 flex-1">
            <div class="flex items-center gap-2 mb-1">
              <span class="font-medium truncate" :class="themeStore.isDark ? 'text-gray-200' : 'text-gray-900'">{{ plan.name }}</span>
              <span class="badge text-xs" :class="getPlanStatusBadgeClass(plan)">{{ getPlanStatusLabel(plan) }}</span>
            </div>
            <p v-if="plan.description" class="text-xs text-themed-muted mb-2">{{ plan.description }}</p>

            <div class="grid grid-cols-2 md:grid-cols-4 gap-2 text-xs">
              <div>
                <span class="text-themed-muted">{{ t('admin.packages.cpu') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ plan.cpu }}%</span>
              </div>
              <div>
                <span class="text-themed-muted">{{ t('admin.packages.memory') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ formatMemory(plan.memory) }}</span>
              </div>
              <div>
                <span class="text-themed-muted">{{ t('admin.packages.disk') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ formatDisk(plan.disk) }}</span>
              </div>
              <div>
                <span class="text-themed-muted">{{ t('resources.plans.price') }}:</span>
                <span class="ml-1 font-medium" :class="themeStore.isDark ? 'text-green-400' : 'text-green-600'">¥{{ formatPrice(plan.price) }}/{{ getBillingCycleLabel(plan.billingCycle) }}</span>
              </div>
            </div>

            <div class="grid grid-cols-2 md:grid-cols-4 gap-2 text-xs mt-2">
              <div>
                <span class="text-themed-muted">{{ t('resources.plans.portLimit') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ plan.portLimit }}</span>
              </div>
              <div>
                <span class="text-themed-muted">{{ t('resources.plans.snapshotLimit') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ plan.snapshotLimit }}</span>
              </div>
              <div v-if="pkg.instance_type !== 'vm'">
                <span class="text-themed-muted">{{ t('resources.plans.swapSize') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ plan.swapSize > 0 ? `${plan.swapSize} MB` : '-' }}</span>
              </div>
              <div>
                <span class="text-themed-muted">{{ t('resources.plans.siteLimit') }}:</span>
                <span class="ml-1 font-mono" :class="themeStore.isDark ? 'text-gray-300' : 'text-gray-700'">{{ plan.siteLimit }}</span>
              </div>
            </div>

            <div class="mt-2 flex flex-wrap gap-x-4 gap-y-1 text-xs">
              <div v-if="plan.slaGuarantee !== null">
                <span class="text-themed-muted">{{ t('resources.plans.slaGuarantee') }}:</span>
                <span class="ml-1 font-medium" :class="themeStore.isDark ? 'text-green-400' : 'text-green-600'">{{ plan.slaGuarantee }}%</span>
              </div>
              <div v-if="plan.trafficResetEnabled">
                <span class="text-themed-muted">{{ t('resources.plans.trafficReset') }}:</span>
                <span class="ml-1 font-medium" :class="themeStore.isDark ? 'text-cyan-300' : 'text-cyan-700'">
                  ¥{{ formatPrice(plan.trafficResetPrice) }}/{{ t('resources.plans.perReset') }}
                </span>
              </div>
            </div>
          </div>

          <div class="flex items-center gap-1 self-end sm:self-start">
            <button
              type="button"
              class="p-2 rounded-lg transition-colors"
              :class="themeStore.isDark ? 'hover:bg-gray-700 text-gray-400 hover:text-blue-400' : 'hover:bg-gray-200 text-gray-500 hover:text-blue-600'"
              :title="t('common.edit')"
              @click="openEditPlanModal(plan)"
            >
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
              </svg>
            </button>
            <button
              type="button"
              class="p-2 rounded-lg transition-colors"
              :class="themeStore.isDark ? 'hover:bg-red-500/20 text-gray-400 hover:text-red-400' : 'hover:bg-red-50 text-gray-500 hover:text-red-500'"
              :title="t('common.delete')"
              @click="deletePlan(plan)"
            >
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
              </svg>
            </button>
          </div>
        </div>
      </div>
    </div>

    <Teleport to="body">
      <Transition name="modal">
        <div v-if="showPlanFormModal" class="modal-overlay">
          <div class="modal-backdrop" @click="showPlanFormModal = false"></div>
          <div class="modal-content max-w-2xl">
            <div class="modal-header">
              <div class="flex items-center gap-3">
                <div class="w-10 h-10 rounded-xl flex items-center justify-center" :class="themeStore.isDark ? 'bg-indigo-500/20' : 'bg-indigo-100'">
                  <svg class="w-5 h-5" :class="themeStore.isDark ? 'text-indigo-400' : 'text-indigo-600'" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2" />
                  </svg>
                </div>
                <div>
                  <h3 class="modal-title">{{ editingPlan ? t('resources.plans.edit') : t('resources.plans.create') }}</h3>
                  <p class="text-xs text-themed-muted mt-0.5">{{ pkg.name }}</p>
                </div>
              </div>
              <button type="button" class="p-1.5 rounded-lg transition-colors" :class="themeStore.isDark ? 'hover:bg-gray-700' : 'hover:bg-gray-100'" @click="showPlanFormModal = false">
                <svg class="w-5 h-5 text-themed-muted" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                </svg>
              </button>
            </div>

            <div class="modal-body space-y-4 max-h-[70vh] overflow-y-auto">
              <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
                <div>
                  <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.name') }} *</label>
                  <input v-model="planForm.name" type="text" class="input" :placeholder="t('resources.plans.namePlaceholder')" />
                </div>
                <div>
                  <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.description') }}</label>
                  <input v-model="planForm.description" type="text" class="input" :placeholder="t('resources.plans.descriptionPlaceholder')" />
                </div>
              </div>
              <p class="text-xs text-orange-500 dark:text-orange-400 -mt-2">{{ t('common.noIncudalHint') }}</p>

              <div class="rounded-xl p-4" :class="themeStore.isDark ? 'bg-gray-800/50 border border-gray-700' : 'bg-gray-50 border border-gray-200'">
                <h4 class="text-sm font-medium mb-3" :class="themeStore.isDark ? 'text-gray-200' : 'text-gray-700'">{{ t('resources.plans.resourceConfig') }}</h4>
                <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">CPU (%)</label>
                    <input v-model.number="planForm.cpu" type="number" min="15" max="10000" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('admin.packages.memory') }} (MB)</label>
                    <input v-model.number="planForm.memory" type="number" min="128" max="62144" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('admin.packages.disk') }} (MB)</label>
                    <input v-model.number="planForm.disk" type="number" min="512" max="104857600" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.portLimit') }}</label>
                    <input v-model.number="planForm.portLimit" type="number" min="0" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.snapshotLimit') }}</label>
                    <input v-model.number="planForm.snapshotLimit" type="number" min="0" class="input" />
                  </div>
                  <div v-if="pkg.instance_type !== 'vm'">
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.swapSize') }} (MB)</label>
                    <input v-model.number="planForm.swapSize" type="number" min="0" max="1048576" step="128" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.siteLimit') }}</label>
                    <input v-model.number="planForm.siteLimit" type="number" min="0" class="input" />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.trafficLimit') }} (GB) *</label>
                    <input v-model="planForm.trafficLimit" type="number" min="1" max="100000" step="1" class="input" required />
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.trafficSpeed') }} (Mbps) *</label>
                    <input v-model.number="planForm.trafficLimitSpeed" type="number" min="1" max="10000" step="1" class="input" required />
                  </div>
                </div>
              </div>

              <div class="rounded-xl p-4" :class="themeStore.isDark ? 'bg-gray-800/50 border border-gray-700' : 'bg-gray-50 border border-gray-200'">
                <h4 class="text-sm font-medium mb-3" :class="themeStore.isDark ? 'text-gray-200' : 'text-gray-700'">{{ t('resources.plans.billingConfig') }}</h4>
                <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.price') }} ({{ t('resources.plans.priceUnit') }})</label>
                    <input v-model.number="planForm.price" type="number" min="0" :max="MAX_PACKAGE_PLAN_PRICE" step="0.01" class="input" />
                    <p class="mt-1 text-xs text-themed-muted">{{ t('resources.plans.priceRangeHint', { max: MAX_PACKAGE_PLAN_PRICE.toFixed(2) }) }}</p>
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.billingCycle') }}</label>
                    <select v-model.number="planForm.billingCycle" class="input">
                      <option v-for="opt in billingCycleOptions" :key="opt.value" :value="opt.value">{{ opt.label }}</option>
                    </select>
                  </div>
                  <div>
                    <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.slaGuarantee') }} (%)</label>
                    <input v-model.number="planForm.slaGuarantee" type="number" min="1" max="100" step="0.01" class="input" placeholder="99.9" />
                  </div>
                </div>
                <div
                  class="mt-4 rounded-lg border p-3"
                  :class="themeStore.isDark ? 'border-gray-700 bg-gray-900/40' : 'border-gray-200 bg-white'"
                >
                  <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
                    <label class="flex min-w-0 cursor-pointer items-start gap-3">
                      <input
                        v-model="planForm.trafficResetEnabled"
                        type="checkbox"
                        class="mt-0.5 h-4 w-4 flex-shrink-0 rounded border-gray-300 text-indigo-600 focus:ring-indigo-500"
                      />
                      <span class="min-w-0">
                        <span class="block text-sm font-medium" :class="themeStore.isDark ? 'text-gray-200' : 'text-gray-800'">
                          {{ t('resources.plans.allowTrafficReset') }}
                        </span>
                        <span class="mt-1 block text-xs leading-5 text-themed-muted">
                          {{ t('resources.plans.allowTrafficResetHint') }}
                        </span>
                      </span>
                    </label>
                    <div v-if="planForm.trafficResetEnabled" class="w-full sm:w-48">
                      <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.trafficResetPrice') }} ({{ t('resources.plans.priceUnit') }})</label>
                      <input
                        v-model.number="planForm.trafficResetPrice"
                        type="number"
                        min="0"
                        :max="MAX_PACKAGE_PLAN_PRICE"
                        step="0.01"
                        class="input"
                      />
                    </div>
                  </div>
                  <p v-if="planForm.trafficResetEnabled" class="mt-2 text-xs leading-5 text-themed-muted sm:ml-7">
                    {{ t('resources.plans.trafficResetPriceHint') }}
                  </p>
                </div>
              </div>

              <div class="grid gap-4 sm:grid-cols-3 sm:items-start">
                <div class="min-w-0 sm:col-span-2">
                  <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.status') }}</label>
                  <div
                    class="h-10 rounded-lg border p-0.5"
                    :class="themeStore.isDark ? 'border-gray-700 bg-gray-900/60' : 'border-gray-200 bg-gray-100/70'"
                  >
                    <div class="grid h-full grid-cols-3 gap-0.5">
                      <button
                        v-for="option in planStatusOptions"
                        :key="option.value"
                        type="button"
                        class="flex h-full min-w-0 items-center justify-center gap-2 rounded-md px-2 text-sm font-medium transition-all"
                        :class="getPlanStatusSegmentClass(option.value)"
                        :aria-pressed="planForm.status === option.value"
                        @click="planForm.status = option.value"
                      >
                        <span class="h-2 w-2 flex-shrink-0 rounded-full" :class="getPlanStatusDotClass(option.value)"></span>
                        <span class="truncate">{{ option.label }}</span>
                      </button>
                    </div>
                  </div>
                  <p class="mt-2 min-h-[2.5rem] text-xs leading-5 text-themed-muted">
                    {{ selectedPlanStatusOption.description }}
                  </p>
                </div>
                <div class="min-w-0">
                  <label class="block text-xs font-medium text-themed-muted mb-1.5">{{ t('resources.plans.sortOrder') }}</label>
                  <input v-model.number="planForm.sortOrder" type="number" min="0" class="input h-10 w-full text-center" />
                </div>
              </div>
            </div>

            <div class="modal-footer">
              <button type="button" class="btn-secondary" @click="showPlanFormModal = false">{{ t('common.cancel') }}</button>
              <button type="button" :disabled="!planForm.name.trim() || planFormLoading" class="btn-primary" @click="savePlan">
                <span v-if="planFormLoading" class="loading-spinner w-4 h-4"></span>
                <template v-else>
                  <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                  </svg>
                  {{ t('common.save') }}
                </template>
              </button>
            </div>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>
