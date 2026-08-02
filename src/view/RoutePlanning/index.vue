<template>
    <div class="routePlanningContainer">
        <div class="leftContainer">
            <div class="formContainer">
                <!-- <h3 class="title">{{ t('routePlanning.title') }}</h3> -->

                <section class="formRow">
                    <b-field :label="t('routePlanning.departureDate')">
                        <b-datepicker
                            class="departureDatePicker"
                            v-model="departureDate"
                            size="is-small"
                            :placeholder="t('routePlanning.datePlaceholder')"
                            :min-date="departureDateMin"
                            :max-date="departureDateMax"
                            :date-formatter="formatDepartureDateForDisplay"
                            :mobile-native="false"
                            trap-focus
                        />
                    </b-field>
                </section>

                <section class="formRow">
                    <b-field :label="t('routePlanning.startPoint')">
                        <b-radio v-for="point in routePoints" :key="`start-${point.value}`" v-model="startPoint"
                            name="start-point" :native-value="point.value" type="is-dark" size="is-small" class="radioItem">
                            {{ point.label }}
                        </b-radio>
                    </b-field>
                </section>

                <section class="formRow">
                    <b-field :label="t('routePlanning.endPoint')">
                        <b-radio v-for="point in routePoints" :key="`end-${point.value}`" v-model="endPoint"
                            name="end-point" :native-value="point.value" type="is-dark" size="is-small" class="radioItem">
                            {{ point.label }}
                        </b-radio>
                    </b-field>
                </section>

                <b-button class="submitBtn" type="is-dark" size="is-small" @click="submitForm">
                    {{ t('common.button.submitAnalysis') }}
                </b-button>
            </div>

            <section class="taskListSection">
                <h4 class="taskListTitle">{{ t('routePlanning.taskListTitle') }}</h4>
                <p v-if="!taskList.length" class="taskEmptyHint">
                    {{ t('routePlanning.taskListEmpty') }}
                </p>
                <div v-else class="taskList">
                    <article v-for="task in taskList" :key="task.taskId" class="taskItem">
                        <p class="taskLine">
                            <span><span style="margin-right:10px">ID:#{{ task.taskId }}</span>{{ formatDateTime(task.createdAt) }}</span>
                            <span :class="statusClass(task.status)">
                                {{ taskStatusText(task.status) }}
                            </span>
                        </p>
                        <p class="taskLine">
                            {{ task.startPort }} -> {{ task.endPort }}
                        </p>
                        <b-button size="is-small" type="is-dark" :disabled="task.status !== 'COMPLETED'"
                            @click.stop="loadTaskResult(task.taskId)">
                            {{ t('routePlanning.loadResult') }}
                        </b-button>
                    </article>
                </div>
            </section>
        </div>

        <div class="resultContainer">
            <h3 class="title">{{ t('routePlanning.resultTitle') }}</h3>
            <p class="summary" v-if="routeSummary">
                <b-button
                    size="is-small"
                    style="margin-right: 10px;"
                    outlined
                    :disabled="!canClearRenderedResult"
                    @click="clearRenderedResult"
                >
                    {{ t('common.button.clearSelection') }}
                </b-button>{{ routeSummary }}
            </p>
            <p class="summary" v-else>
                {{ t('routePlanning.emptyHint') }}
            </p>

            <div class="visualLayout">
                <div class="globePanel">
                    <RoutePhere
                        ref="routePhereRef"
                        :route-ports="ROUTE_PORTS"
                        :selected-start-point="startPoint"
                        :selected-end-point="endPoint"
                        :route-result="activeTask?.result || null"
                        :sea-ice-frames="normalizedSeaIceFrames"
                        :local-risk-frames="normalizedLocalRiskFrames"
                        class="sphereBox"
                        @playback-change="handlePlaybackChange"
                    />

                    <section v-if="activeTask && activeTask.result" class="resultMeta">
                        <div class="metaGrid">
                            <article class="metaCard" v-for="item in resultMetaCards" :key="item.key">
                                <p class="metaLabel">{{ item.label }}</p>
                                <p class="metaValue">{{ item.value }}</p>
                            </article>
                        </div>

                        <p v-if="activeTask.error" class="taskError">
                            {{ t('routePlanning.meta.error') }}: {{ activeTask.error }}
                        </p>
                    </section>
                </div>

                <aside v-if="hasResult" class="sidePanels">
                    <section class="riskPanel riskPanelTop">
                        <div class="panelHeader">
                            <div>
                                <p class="panelTitle">{{ t('routePlanning.riskImagesTitle') }}</p>
                                <p class="panelSubTitle">{{ currentRiskImageTitle }}</p>
                            </div>
                            <b-button
                                size="is-small"
                                type="is-white"
                                outlined
                                :disabled="!riskImagesDownloadUrl"
                                @click="downloadByUrl(riskImagesDownloadUrl)"
                            >
                                {{ t('routePlanning.downloadRiskImages') }}
                            </b-button>
                        </div>

                        <div v-if="normalizedRiskImages.length" class="riskCarouselWrap">
                            <b-carousel v-model="activeRiskImageIndex" :autoplay="false" :pause-hover="true" progress-type="is-dark" pause-text="" :arrow="false">
                                <b-carousel-item v-for="(item, index) in normalizedRiskImages" :key="`${item.title}-${index}`">
                                    <section>
                                        <div class="riskImageCard" @click="openRiskImagePreview(item)">
                                            <img :src="item.imageUrl" :alt="item.title">
                                            <p>{{ item.title }}</p>
                                        </div>
                                    </section>
                                </b-carousel-item>
                            </b-carousel>
                        </div>
                        <p v-else class="panelEmpty">{{ t('routePlanning.riskImagesEmpty') }}</p>
                    </section>

                    <section class="riskPanel riskPanelBottom">
                        <div class="panelHeader">
                            <div>
                                <p class="panelTitle">{{ t('routePlanning.localRiskTitle') }}</p>
                                <p class="panelSubTitle">{{ currentPlaybackDateLabel || '-' }}</p>
                            </div>
                            <b-button
                                size="is-small"
                                 type="is-white"
                                outlined
                                :disabled="!localRiskImagesDownloadUrl"
                                @click="downloadByUrl(localRiskImagesDownloadUrl)"
                            >
                                {{ t('routePlanning.downloadLocalRiskImages') }}
                            </b-button>
                        </div>

                        <div v-if="currentLocalRiskFrame?.imageUrl" class="localRiskCard">
                            <img :src="currentLocalRiskFrame.imageUrl" :alt="currentLocalRiskFrame.title || t('routePlanning.localRiskTitle')">
                            <div class="localRiskMeta">
                                <!-- <p><strong>{{ t('routePlanning.localRiskCurrentTitle') }}：</strong>{{ currentLocalRiskFrame.title || '-' }}</p> -->
                                <p><strong>{{ t('routePlanning.localRiskCoordinates') }}：</strong>{{ currentLocalRiskFrame.coordsText || '-' }}</p>
                            </div>
                        </div>
                        <div v-else class="localRiskEmptyState">
                            <p class="panelEmpty">{{ t('routePlanning.localRiskEmpty') }}</p>
                            <p class="panelEmptyHint">{{ currentPlaybackDateLabel || '-' }}</p>
                        </div>
                    </section>
                </aside>
            </div>
        </div>
    </div>

    <b-modal v-model="isRiskPreviewModalActive">
        <div v-if="riskPreviewImage" class="riskPreviewModal">
            <img :src="riskPreviewImage.imageUrl" :alt="riskPreviewImage.title">
            <p>{{ riskPreviewImage.title }}</p>
        </div>
    </b-modal>
</template>

<script setup>
import RoutePhere from './components/RoutePhere.vue'
import { useI18n } from 'vue-i18n'
import { errorToast, openToast } from '@/utils/toast'
import { ROUTE_PORTS } from '@/constants/routePorts'
import { useRoutePlanningStore } from '@/store'

const { t, te } = useI18n()
const routePlanningStore = useRoutePlanningStore()
const websiteUrl = import.meta.env.VITE_WEBSITE_URL || ''
const downloadBaseUrl = import.meta.env.VITE_DOWNLOAD_BASE_URL || ''

const startPoint = ref('')
const endPoint = ref('')
const departureDate = ref(null)
const routePhereRef = ref(null)
const hasResult = ref(false)
const activeRiskImageIndex = ref(0)
const currentLocalRiskFrame = ref(null)
const currentPlaybackDateLabel = ref('')
const riskPreviewImage = ref(null)
const isRiskPreviewModalActive = ref(false)

const departureDateMin = new Date(2025, 6, 1)
const departureDateMax = new Date(2025, 7, 31)
const taskList = computed(() => routePlanningStore.taskList)
const activeTask = computed(() => routePlanningStore.activeTask)
const routeResultData = computed(() => activeTask.value?.result || null)
const canClearRenderedResult = computed(() => !!activeTask.value?.result)

const getPortLabel = (portName) => {
    const key = `routePlanning.portLabels.${portName}`
    return te(key) ? t(key) : portName
}

const routePoints = computed(() => ROUTE_PORTS.map((port) => ({
    value: port.name,
    label: getPortLabel(port.name)
})))

const formatDateTime = (value) => {
    if (!value) return '-'
    const date = new Date(value)
    if (Number.isNaN(date.getTime())) return '-'
    return date.toLocaleString()
}

const formatDepartureDate = (date) => {
    if (!(date instanceof Date) || Number.isNaN(date.getTime())) return ''
    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${year}-${month}-${day}`
}

// 仅控制选择器输入框的显示；v-model 仍保留完整 Date，提交时仍使用 YYYY-MM-DD。
const formatDepartureDateForDisplay = (date) => {
    if (!(date instanceof Date) || Number.isNaN(date.getTime())) return ''
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${month}-${day}`
}

const isDepartureDateInRange = (date) => {
    if (!(date instanceof Date) || Number.isNaN(date.getTime())) return false
    return date >= departureDateMin && date <= departureDateMax
}

const normalizedDepartureDate = computed(() => {
    if (!(departureDate.value instanceof Date)) return null
    return isDepartureDateInRange(departureDate.value) ? departureDate.value : null
})

const normalizeAssetUrl = (url) => {
    if (!url || typeof url !== 'string') return ''
    if (/^https?:\/\//i.test(url)) return url
    if (url.startsWith('/')) return `${websiteUrl}${url}`
    return `${websiteUrl}/${url}`
}

const normalizeApiUrl = (url) => {
    if (!url || typeof url !== 'string') return ''
    if (/^https?:\/\//i.test(url)) return url
    const base = downloadBaseUrl.trim().replace(/\/$/, '')
    const path = url.startsWith('/') ? url : `/${url}`
    return base ? `${base}${path}` : path
}

const normalizeDateValue = (value) => {
    if (!value) return null
    if (value instanceof Date && !Number.isNaN(value.getTime())) return value
    const parsed = new Date(value)
    if (Number.isNaN(parsed.getTime())) return null
    return parsed
}

const formatDateKey = (value) => {
    const date = normalizeDateValue(value)
    if (!date) return ''
    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${year}-${month}-${day}`
}

const formatDateLabel = (value) => {
    const date = normalizeDateValue(value)
    if (!date) return ''
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${month}-${day}`
}

const resolveFrameDateValue = (item) => {
    return item?.date || item?.forecast_date || item?.voyage_date || item?.current_date || item?.time || item?.datetime || ''
}

const formatCoordValue = (value) => {
    const numericValue = Number(value)
    if (!Number.isFinite(numericValue)) return '-'
    return numericValue.toFixed(4)
}

const normalizedSeaIceFrames = computed(() => {
    const items = routeResultData.value?.sea_ice_images
    if (!Array.isArray(items)) return []

    return items.map((item) => {
        const rawDate = resolveFrameDateValue(item)
        const rectangleDegrees = item?.rectangle_degrees || item?.rectangle || item?.cesium_rectangle
        return {
            rawDate,
            dateKey: formatDateKey(rawDate),
            dateLabel: formatDateLabel(rawDate),
            cesiumUrl: normalizeAssetUrl(item?.cesium_image_url || item?.url || item?.path || item?.image_url || ''),
            rectangle: Array.isArray(rectangleDegrees)
                ? rectangleDegrees
                : rectangleDegrees
                    ? [rectangleDegrees.west, rectangleDegrees.south, rectangleDegrees.east, rectangleDegrees.north]
                    : undefined,
            west: item?.west,
            south: item?.south,
            east: item?.east,
            north: item?.north,
            alpha: item?.alpha
        }
    }).filter((item) => item.dateKey && item.cesiumUrl)
})

const normalizedRiskImages = computed(() => {
    const items = routeResultData.value?.risk_images
    if (!Array.isArray(items)) return []

    return items.map((item, index) => ({
        index,
        title: item?.name || item?.title || `${t('routePlanning.riskImageFallbackTitle')} ${index + 1}`,
        imageUrl: normalizeAssetUrl(item?.url || item?.image_url || item?.path || '')
    })).filter((item) => item.imageUrl)
})

const normalizedLocalRiskFrames = computed(() => {
    const items = routeResultData.value?.local_risk_image_url
    if (!Array.isArray(items)) return []

    return items.map((item) => {
        const rawDate = resolveFrameDateValue(item)
        const lat = item?.lat ?? item?.latitude ?? item?.coord_lat ?? item?.y
        const lon = item?.lon ?? item?.lng ?? item?.longitude ?? item?.coord_lon ?? item?.x
        return {
            rawDate,
            dateKey: formatDateKey(rawDate),
            dateLabel: formatDateLabel(rawDate),
            title: item?.title || item?.name || '',
            imageUrl: normalizeAssetUrl(item?.image_url || item?.url || item?.path || item?.cesium_image_url || ''),
            lat,
            lon,
            coordsText: `Lat ${formatCoordValue(lat)}, Lon ${formatCoordValue(lon)}`
        }
    }).filter((item) => item.dateKey && item.imageUrl)
})

const riskImagesDownloadUrl = computed(() => normalizeApiUrl(routeResultData.value?.risk_images_download_url || ''))
const localRiskImagesDownloadUrl = computed(() => normalizeApiUrl(routeResultData.value?.local_risk_images_download_url || ''))

const currentRiskImageTitle = computed(() => {
    const current = normalizedRiskImages.value[activeRiskImageIndex.value]
    return current?.title || t('routePlanning.riskImagesEmpty')
})

const routeSummary = computed(() => {
    if (!hasResult.value || !activeTask.value) return ''
    return t('routePlanning.taskSummary', {
        taskId: activeTask.value.taskId,
        status: taskStatusText(activeTask.value.status)
    })
})

const formatEtaText = (hours) => {
    const value = Number(hours)
    if (!Number.isFinite(value)) return '-'
    return t('routePlanning.meta.etaHoursValue', { value: value.toFixed(1) })
}

const formatDistanceNmText = (nm) => {
    const value = Number(nm)
    if (!Number.isFinite(value)) return '-'
    return t('routePlanning.meta.distanceNmValue', { value: value.toFixed(1) })
}

const resultStartPortText = computed(() => {
    const result = routeResultData.value
    const portName = result?.start_port || activeTask.value?.startPort || '-'
    return portName === '-' ? '-' : getPortLabel(portName)
})

const resultEndPortText = computed(() => {
    const result = routeResultData.value
    const portName = result?.end_port || activeTask.value?.endPort || '-'
    return portName === '-' ? '-' : getPortLabel(portName)
})

const resultMetaCards = computed(() => {
    const result = routeResultData.value
    if (!result) return []

    return [
        {
            key: 'startPort',
            label: t('routePlanning.meta.startPort'),
            value: resultStartPortText.value
        },
        {
            key: 'endPort',
            label: t('routePlanning.meta.endPort'),
            value: resultEndPortText.value
        },
        {
            key: 'departureDate',
            label: t('routePlanning.meta.departureDate'),
            value: result?.departure_date || '-'
        },
        {
            key: 'etaHours',
            label: t('routePlanning.meta.etaHours'),
            value: formatEtaText(result?.eta_hours)
        },
        {
            key: 'distanceNm',
            label: t('routePlanning.meta.distanceNm'),
            value: formatDistanceNmText(result?.distance_nm)
        },
        {
            key: 'route',
            label: t('routePlanning.meta.route'),
            value: `${resultStartPortText.value} -> ${resultEndPortText.value}`
        }
    ]
})

const taskStatusText = (status) => {
    if (status === 'COMPLETED') return t('routePlanning.taskStatus.completed')
    if (status === 'FAILED') return t('routePlanning.taskStatus.failed')
    if (status === 'TIMEOUT') return t('routePlanning.taskStatus.timeout')
    return t('routePlanning.taskStatus.inProgress')
}

const statusClass = (status) => {
    if (status === 'COMPLETED') return 'statusSuccess'
    if (status === 'FAILED' || status === 'TIMEOUT') return 'statusFailed'
    return 'statusProcessing'
}

const loadTaskResult = async (taskId) => {
    const task = taskList.value.find((item) => item.taskId === taskId)
    if (!task || task.status !== 'COMPLETED') return
    routePlanningStore.setActiveTask(task.taskId)
    await nextTick()
    hasResult.value = true
    if (normalizedRiskImages.value.length) {
        activeRiskImageIndex.value = 0
    }
    if (normalizedLocalRiskFrames.value.length) {
        currentLocalRiskFrame.value = normalizedLocalRiskFrames.value[0]
        currentPlaybackDateLabel.value = normalizedLocalRiskFrames.value[0].dateLabel || ''
    }
    openToast('routePlanning.loadSuccess')
}

const clearRenderedResult = () => {
    routePhereRef.value?.clearRouteResult?.()
    routePlanningStore.setActiveTask(null)
    hasResult.value = false
    currentLocalRiskFrame.value = null
    currentPlaybackDateLabel.value = ''
    activeRiskImageIndex.value = 0
}

const handlePlaybackChange = (payload = {}) => {
    currentPlaybackDateLabel.value = payload?.dateLabel || ''
    currentLocalRiskFrame.value = payload?.localRiskFrame || null
}

const openRiskImagePreview = (item) => {
    if (!item?.imageUrl) return
    riskPreviewImage.value = item
    isRiskPreviewModalActive.value = true
}

const closeRiskImagePreview = () => {
    isRiskPreviewModalActive.value = false
    riskPreviewImage.value = null
}

const downloadByUrl = (url) => {
    if (!url) return
    window.open(url, '_blank', 'noopener,noreferrer')
}

const submitForm = async () => {
    if (!routePoints.value.length || !startPoint.value || !endPoint.value) {
        errorToast('routePlanning.errors.noPortsAvailable')
        return
    }

    if (!departureDate.value) {
        errorToast('routePlanning.errors.dateRequired')
        return
    }

    if (!normalizedDepartureDate.value) {
        errorToast('routePlanning.errors.dateOutOfRange')
        return
    }

    if (startPoint.value === endPoint.value) {
        errorToast('routePlanning.errors.pointsMustDiffer')
        return
    }

    try {
        await routePlanningStore.submitRoutePlanTask({
            startPort: startPoint.value,
            endPort: endPoint.value,
            departureDate: formatDepartureDate(normalizedDepartureDate.value)
        })
        hasResult.value = false
        currentLocalRiskFrame.value = null
        currentPlaybackDateLabel.value = ''
        activeRiskImageIndex.value = 0
        openToast('routePlanning.submitSuccess')
    } catch (error) {
        errorToast('common.message.uploadFail')
    }
}

onMounted(async () => {
    routePlanningStore.restartPendingPolling()
    if (activeTask.value?.status === 'COMPLETED' && activeTask.value?.result) {
        await nextTick()
        hasResult.value = true
    }
})

onBeforeUnmount(() => {
    routePlanningStore.clearAllPolling()
})

watch(
    () => activeTask.value,
    (task) => {
        if (!task) {
            hasResult.value = false
            currentLocalRiskFrame.value = null
            currentPlaybackDateLabel.value = ''
            return
        }
        hasResult.value = task.status === 'COMPLETED' && !!task.result
    },
    { deep: true, immediate: true }
)

watch(
    () => normalizedRiskImages.value.length,
    (length) => {
        if (!length) {
            activeRiskImageIndex.value = 0
            return
        }
        if (activeRiskImageIndex.value >= length) {
            activeRiskImageIndex.value = 0
        }
    },
    { immediate: true }
)
</script>

<style scoped lang="scss">
$container-height: 560px;

.routePlanningContainer {
    display: flex;
    justify-content: flex-start;
    align-items: stretch;
    width: 100%;
    padding-left: 28px;
    column-gap: 18px;

    .title {
        color: #fff;
        text-align: left;
        font-size: 18px;
        margin-bottom: 14px;
    }

    .leftContainer {
        width: 310px;
        display: flex;
        flex-direction: column;
        gap: 14px;
        flex-shrink: 0;
    }

    .formContainer {
        width: 100%;
        padding: 18px;
        border-radius: 10px;
        background-color: var(--bulma-scheme-main);
        flex-shrink: 0;
        overflow-y: auto;
        overflow-x: hidden;

        .formRow {
            margin-bottom: 14px;
        }

        // 可选日期范围固定在 2025 年，日历弹层无需向用户展示年份。
        :deep(.departureDatePicker .datepicker-header .pagination-list .field > .control:nth-child(2)) {
            display: none;
        }

        .radioItem {
            margin-right: 16px;
            margin-bottom: 8px;
            min-width: 120px;
        }

        .submitBtn {
            margin-top: 8px;
        }
    }

    .taskListSection {
        height: 230px;
        border-radius: 10px;
        background: rgba(10, 18, 26, 0.45);
        border: 1px solid rgba(255, 255, 255, 0.12);
        padding: 12px;
        display: flex;
        flex-direction: column;
        overflow: hidden;

        .taskListTitle {
            color: #fff;
            font-size: 14px;
            margin-bottom: 8px;
        }

        .taskEmptyHint {
            color: #b9c0c8;
            font-size: 12px;
            margin-bottom: 0;
        }

        .taskList {
            flex: 1;
            overflow-y: auto;
            overflow-x: hidden;
            padding-right: 4px;
        }

        .taskList::-webkit-scrollbar {
            width: 6px;
        }

        .taskList::-webkit-scrollbar-thumb {
            background: rgba(255, 255, 255, 0.24);
            border-radius: 99px;
        }

        .taskList::-webkit-scrollbar-track {
            background: transparent;
        }

        .taskItem {
            border: 1px solid rgba(255, 255, 255, 0.18);
            border-radius: 8px;
            padding: 7px 8px;
            margin-bottom: 8px;
            background: rgba(10, 18, 26, 0.35);
            cursor: pointer;
            min-height: 72px;

            .taskLine {
                display: flex;
                justify-content: space-between;
                gap: 8px;
                color: #dbe3ec;
                font-size: 11px;
                margin-bottom: 6px;
                line-height: 1.4;
            }

            .statusProcessing {
                color: #f0d286;
            }

            .statusSuccess {
                color: #79d89e;
            }

            .statusFailed {
                color: #ff9a9a;
            }

            :deep(.button.is-small) {
                height: 24px;
                font-size: 11px;
                padding: 0 10px;
            }
        }
    }

    .resultContainer {
        flex: 1;
        min-width: 0;
        height: $container-height;
        padding: 0;
        background-color: transparent;
        border-radius: 0;

        .summary {
            display: flex;
            align-items: center;
            color: #fff;
            min-height: 24px;
            margin-bottom: 10px;
            font-size: 14px;
        }

        .visualLayout {
            display: flex;
            gap: 14px;
            min-height: 410px;
            align-items: flex-start;
        }

        .globePanel {
            flex: 1;
            min-width: 0;
            display: flex;
            flex-direction: column;
            padding: 10px 12px;
            border-radius: 10px;
            background: rgba(8, 13, 20, 0.42);
            border: 1px solid rgba(255, 255, 255, 0.12);
        }

        .sphereBox {
            flex: 0 0 auto;
            min-width: 0;
            height: 320px;
            border-radius: 8px;
        }

        .sidePanels {
            width: 420px;
            height: 560px;
            display: flex;
            flex-direction: column;
            gap: 12px;
            flex-shrink: 0;
        }

        .riskPanel {
            flex: 1;
            border-radius: 10px;
            background: rgba(8, 13, 20, 0.58);
            border: 1px solid rgba(255, 255, 255, 0.12);
            padding: 12px;
            min-height: 0;
            overflow: hidden;
        }

        .riskPanelTop,
        .riskPanelBottom {
            height: 0;
        }

        .panelHeader {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
            margin-bottom: 10px;
        }

        .panelTitle {
            color: #fff;
            font-size: 13px;
            font-weight: 600;
            margin-bottom: 2px;
        }

        .panelSubTitle {
            color: #aebdcd;
            font-size: 11px;
            margin-bottom: 0;
            line-height: 1.4;
        }

        .panelEmpty,
        .panelEmptyHint {
            color: #9fb0c2;
            font-size: 12px;
            margin-bottom: 0;
        }

        .riskCarouselWrap {
            height: calc(100% - 46px);

            :deep(.carousel) {
                height: 100%;
            }

            :deep(.carousel-items) {
                height: 100%;
            }

            :deep(.carousel-item) {
                height: 100%;
            }
        }

        .riskImageCard,
        .localRiskCard {
            height: 100%;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 10px;
        }

        .riskImageCard {
            cursor: zoom-in;
        }

        .riskImageCard img {
            max-width: calc(100% - 200px);
            max-height: calc(100% - 50px);
            object-fit: contain;
            border-radius: 6px;
            background: rgba(255, 255, 255, 0.04);
        }
          .localRiskCard img {
            max-width: calc(100% - 20px);
            max-height: calc(100% - 50px);
            object-fit: contain;
            border-radius: 6px;
            background: rgba(255, 255, 255, 0.04);
        }


        .riskImageCard p {
            color: #dbe3ec;
            font-size: 12px;
            margin-bottom: 0;
            text-align: center;
            line-height: 1.4;
        }

        .localRiskMeta {
            width: 100%;

            p {
                color: #dbe3ec;
                font-size: 12px;
                line-height: 1.5;
                margin-bottom: 4px;
                word-break: break-word;
            }
        }

        .localRiskEmptyState {
            height: calc(100% - 46px);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            gap: 8px;
        }

        .resultMeta {
            margin-top: 10px;
            padding: 10px 12px;
            border-radius: 8px;
            background: rgba(8, 13, 20, 0.42);

            .metaGrid {
                display: grid;
                grid-template-columns: repeat(2, minmax(0, 1fr));
                gap: 10px;
            }

            .metaCard {
                padding: 8px 10px;
                border-radius: 6px;
                border: 1px solid rgba(255, 255, 255, 0.12);
                background: rgba(19, 30, 43, 0.45);
            }

            .metaLabel {
                color: #9fb0c2;
                font-size: 11px;
                margin-bottom: 4px;
            }

            .metaValue {
                color: #dbe3ec;
                font-size: 13px;
                line-height: 1.5;
                margin-bottom: 0;
                word-break: break-word;
            }

            .taskError {
                color: #ffb4b4;
                font-size: 12px;
                line-height: 1.6;
                margin-top: 8px;
                margin-bottom: 0;
            }
        }
    }
}

.riskPreviewModal {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    padding: 12px;

    img {
        max-width: min(80vw, 960px);
        max-height: 75vh;
        object-fit: contain;
        border-radius: 8px;
        background: rgba(255, 255, 255, 0.04);
    }

    p {
        color: #fff;
        font-size: 14px;
        line-height: 1.5;
        margin-bottom: 0;
        text-align: center;
    }
}

@media screen and (max-width: 1200px) {
    .routePlanningContainer {
        .resultContainer {
            .visualLayout {
                flex-direction: column;
            }

            .sidePanels {
                width: 100%;
                display: grid;
                grid-template-columns: repeat(2, minmax(0, 1fr));
            }
        }
    }
}

@media screen and (max-width: 1024px) {
    .routePlanningContainer {
        flex-direction: column;
        padding-left: 0;
        width: 100%;

        .leftContainer {
            width: 100%;
            display: flex;
            flex-direction: row;
            gap: 10px;
            margin: 10px 0;
        }

        .formContainer,
        .taskListSection {
            flex: 1;
            width: auto;
        }
    }
}

@media screen and (max-width: 760px) {
    .routePlanningContainer {
        .leftContainer {
            flex-direction: column;
        }

        .resultContainer {
            .sidePanels {
                grid-template-columns: minmax(0, 1fr);
            }

            .resultMeta {
                .metaGrid {
                    grid-template-columns: minmax(0, 1fr);
                }
            }
        }
    }
}
</style>
