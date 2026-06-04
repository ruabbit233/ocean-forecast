<template>
	<div class="routeSphereContainer">
		<div ref="cesiumContainerRef" class="cesiumContainer"></div>
		<div v-if="currentDateLabel" class="currentDateBadge">
			{{ currentDateLabel }}
		</div>
	</div>
</template>

<script setup>
import * as Cesium from 'cesium'

const props = defineProps({
	initialCoords: { type: Array, default: () => [-180, 90, 180, 90] },
	routePorts: { type: Array, default: () => [] },
	selectedStartPoint: { type: String, default: '' },
	selectedEndPoint: { type: String, default: '' },
	routeResult: { type: Object, default: null },
	seaIceFrames: { type: Array, default: () => [] },
	localRiskFrames: { type: Array, default: () => [] }
})

const emit = defineEmits(['playback-change'])

const viewer = ref(null)
const imageryLayers = ref(null)
const cesiumContainerRef = ref(null)
const seaIceLayer = ref(null)
let creditEl = null
const portEntities = new Map()
const resultImageLayers = ref([])
const routePointEntities = ref([])
const routeLineEntity = ref(null)
const routeEndpointEntities = ref([])
const focusableRouteEntities = ref([])
const hoverFocusedEntity = ref(null)
const selectedFocusedEntity = ref(null)
const activeFocusedEntity = ref(null)
const currentDateLabel = ref('')
let routeFocusHandler = null
let routeAnimationTimer = null
const websiteUrl = import.meta.env.VITE_WEBSITE_URL || ''
const ROUTE_FRAME_INTERVAL = 100
const ROUTE_PROGRESS_SUBSTEPS = 6

const PORT_ENTITY_ID_PREFIX = 'route-port-'
const START_PORT_COLOR = Cesium.Color.fromCssColorString('#2fa35f')
const END_PORT_COLOR = Cesium.Color.fromCssColorString('#2f75c9')
const DEFAULT_PORT_COLOR = Cesium.Color.fromCssColorString('#c95454')

const getPortStyle = (name) => {
	const isStart = name === props.selectedStartPoint
	const isEnd = name === props.selectedEndPoint

	if (isStart) {
		return {
			selected: true,
			color: START_PORT_COLOR
		}
	}

	if (isEnd) {
		return {
			selected: true,
			color: END_PORT_COLOR
		}
	}

	return {
		selected: false,
		color: DEFAULT_PORT_COLOR
	}
}

const clearPortEntities = () => {
	if (!viewer.value) return
	for (const entity of portEntities.values()) {
		viewer.value.entities.remove(entity)
	}
	portEntities.clear()
}

const clearResultLayers = () => {
	if (!imageryLayers.value) return
	for (const layer of resultImageLayers.value) {
		try {
			imageryLayers.value.remove(layer, true)
		} catch (e) {
			// ignore
		}
	}
	resultImageLayers.value = []
}

const stopRoutePlayback = () => {
	if (routeAnimationTimer) {
		clearInterval(routeAnimationTimer)
		routeAnimationTimer = null
	}
}

const clearPlaybackState = () => {
	currentDateLabel.value = ''
	emit('playback-change', {
		index: -1,
		dateKey: '',
		dateLabel: '',
		localRiskFrame: null
	})
}

const clearRouteEntities = () => {
	if (!viewer.value) return
	clearRouteFocusState()
	focusableRouteEntities.value = []
	if (routeLineEntity.value) {
		try {
			viewer.value.entities.remove(routeLineEntity.value)
		} catch (e) {
			// ignore
		}
		routeLineEntity.value = null
	}

	for (const pointEntity of routePointEntities.value) {
		try {
			viewer.value.entities.remove(pointEntity)
		} catch (e) {
			// ignore
		}
	}
	routePointEntities.value = []

	for (const endpointEntity of routeEndpointEntities.value) {
		try {
			viewer.value.entities.remove(endpointEntity)
		} catch (e) {
			// ignore
		}
	}
	routeEndpointEntities.value = []

	stopRoutePlayback()
}

const isEntityFocusable = (entity) => {
	return !!entity && focusableRouteEntities.value.includes(entity)
}

const hideEntityLabel = (entity) => {
	if (!entity?.label) return
	entity.label.show = false
}

const showEntityLabel = (entity) => {
	if (!entity?.label) return
	entity.label.show = true
}

const updateFocusedEntityDisplay = () => {
	const nextActive = hoverFocusedEntity.value || selectedFocusedEntity.value || null
	if (activeFocusedEntity.value && activeFocusedEntity.value !== nextActive) {
		hideEntityLabel(activeFocusedEntity.value)
	}
	if (nextActive) {
		showEntityLabel(nextActive)
	}
	activeFocusedEntity.value = nextActive
}

const clearRouteFocusState = () => {
	hoverFocusedEntity.value = null
	selectedFocusedEntity.value = null
	if (activeFocusedEntity.value) {
		hideEntityLabel(activeFocusedEntity.value)
	}
	activeFocusedEntity.value = null
}

const formatCoordinate = (value) => {
	const numericValue = Number(value)
	if (!Number.isFinite(numericValue)) return '-'
	return numericValue.toFixed(4)
}

const getCoordinateLabelText = ({ lat, lon, title = '' }) => {
	const lines = []
	if (title) lines.push(title)
	lines.push(`Lat: ${formatCoordinate(lat)}`)
	lines.push(`Lon: ${formatCoordinate(lon)}`)
	return lines.join('\n')
}

const getRoutePointTitle = (index) => `Route Point : ${index}`

const createCoordinateDescription = ({ lat, lon, title = '' }) => {
	const titleLine = title ? `<div><strong>${title}</strong></div>` : ''
	return `
			<div style="line-height: 1.7; font-size: 13px;">
				${titleLine}
				<div><strong>Latitude:</strong> ${formatCoordinate(lat)}</div>
				<div><strong>Longitude:</strong> ${formatCoordinate(lon)}</div>
			</div>
		`
}

const setupRouteFocusInteraction = () => {
	if (!viewer.value) return

	if (routeFocusHandler) {
		routeFocusHandler.destroy()
		routeFocusHandler = null
	}

	routeFocusHandler = new Cesium.ScreenSpaceEventHandler(viewer.value.scene.canvas)

	routeFocusHandler.setInputAction((movement) => {
		const picked = viewer.value.scene.pick(movement.endPosition)
		const pickedEntity = picked?.id
		hoverFocusedEntity.value = isEntityFocusable(pickedEntity) ? pickedEntity : null
		updateFocusedEntityDisplay()
	}, Cesium.ScreenSpaceEventType.MOUSE_MOVE)

	routeFocusHandler.setInputAction((movement) => {
		const picked = viewer.value.scene.pick(movement.position)
		const pickedEntity = picked?.id
		selectedFocusedEntity.value = isEntityFocusable(pickedEntity) ? pickedEntity : null
		updateFocusedEntityDisplay()
	}, Cesium.ScreenSpaceEventType.LEFT_CLICK)
}

const resolveAssetUrl = (url) => {
	if (!url || typeof url !== 'string') return ''
	if (/^https?:\/\//i.test(url)) return url
	if (url.startsWith('/')) return `${websiteUrl}${url}`
	return `${websiteUrl}/${url}`
}

const renderPortEntities = () => {
	if (!viewer.value) return

	clearPortEntities()

	for (const port of props.routePorts) {
		if (typeof port?.lat !== 'number' || typeof port?.lon !== 'number' || !port?.name) continue

		const { selected, color } = getPortStyle(port.name)
		const portEntityId = `${PORT_ENTITY_ID_PREFIX}${port.name}`
		const entity = viewer.value.entities.add({
			id: portEntityId,
			name: port.name,
			description: `
					<div style="line-height: 1.7; font-size: 13px;">
						<div><strong>ID:</strong> ${portEntityId}</div>
						<div><strong>Latitude:</strong> ${port.lat}</div>
						<div><strong>Longitude:</strong> ${port.lon}</div>
					</div>
				`,
			position: Cesium.Cartesian3.fromDegrees(port.lon, port.lat),
			point: {
				pixelSize: selected ? 14 : 9,
				color,
				outlineColor: Cesium.Color.WHITE.withAlpha(0.92),
				outlineWidth: selected ? 1.8 : 1.2,
				disableDepthTestDistance: Number.POSITIVE_INFINITY
			}
		})

		portEntities.set(port.name, entity)
	}
}

const resolveRectangle = (item = {}) => {
	if (Array.isArray(item?.rectangle) && item.rectangle.length === 4) {
		return Cesium.Rectangle.fromDegrees(...item.rectangle)
	}

	const corners = [item.west, item.south, item.east, item.north]
	if (corners.every((value) => typeof value === 'number')) {
		return Cesium.Rectangle.fromDegrees(...corners)
	}

	return Cesium.Rectangle.fromDegrees(...props.initialCoords)
}

const normalizeImageItems = (result) => {
	if (!result) return []

	const imageCandidates = result.cesium_images
		|| result.images
		|| result.cesiumImageList
		|| result.cesium_image_urls
		|| result.tiles
		|| []

	if (typeof imageCandidates === 'string') {
		return [{ url: resolveAssetUrl(imageCandidates) }]
	}

	const normalized = Array.isArray(imageCandidates)
		? imageCandidates
			.map((item) => {
				if (typeof item === 'string') {
					return { url: resolveAssetUrl(item) }
				}

				const rectangleDegrees = item?.rectangle_degrees || item?.rectangle
				return {
					url: resolveAssetUrl(item.url || item.path || item.image_url || item.cesium_image_url || ''),
					rectangle: Array.isArray(rectangleDegrees)
						? rectangleDegrees
						: rectangleDegrees
							? [
								rectangleDegrees.west,
								rectangleDegrees.south,
								rectangleDegrees.east,
								rectangleDegrees.north
							]
							: item.rectangle,
					west: item.west,
					south: item.south,
					east: item.east,
					north: item.north,
					alpha: item.alpha
				}
			})
			.filter((item) => !!item.url)
		: []

	const singleCesiumUrl = result.n3125_cesium_image_url
		|| result?.n3125_image?.cesium_image_url
		|| result?.n3125_image?.cesium_meta?.image_path

	if (singleCesiumUrl) {
		const rectangle = result.n3125_cesium_rectangle
			|| result?.n3125_image?.cesium_meta?.rectangle_degrees

		normalized.push({
			url: resolveAssetUrl(singleCesiumUrl),
			rectangle: rectangle
				? [rectangle.west, rectangle.south, rectangle.east, rectangle.north]
				: undefined,
			alpha: 0.86
		})
	}

	return normalized
}

const renderRouteImageLayers = async (result) => {
	if (!imageryLayers.value) return
	clearResultLayers()

	const items = normalizeImageItems(result)
	for (const item of items) {
		try {
			let provider
			if (item.url.includes('{z}') && item.url.includes('{x}') && item.url.includes('{y}')) {
				provider = new Cesium.UrlTemplateImageryProvider({
					url: item.url,
					rectangle: resolveRectangle(item)
				})
			} else {
				provider = await Cesium.SingleTileImageryProvider.fromUrl(item.url, {
					rectangle: resolveRectangle(item)
				})
			}
			const layer = imageryLayers.value.addImageryProvider(provider)
			layer.alpha = typeof item.alpha === 'number' ? item.alpha : 0.82
			resultImageLayers.value.push(layer)
		} catch (e) {
			// ignore single layer errors and keep processing remaining layers
		}
	}
}

const normalizeRoutePoints = (result) => {
	if (!result) return []
	const pointCandidates = result.route_points || result.path_points || result.points || result.path || []
	if (!Array.isArray(pointCandidates)) return []

	return pointCandidates
		.map((point) => {
			if (Array.isArray(point) && point.length >= 2) {
				return { lon: Number(point[0]), lat: Number(point[1]) }
			}

			const lon = Number(point?.lon ?? point?.lng ?? point?.longitude ?? point?.x)
			const lat = Number(point?.lat ?? point?.latitude ?? point?.y)
			if (Number.isNaN(lon) || Number.isNaN(lat)) return null
			return { lon, lat }
		})
		.filter(Boolean)
}

const getPortCoordinateByName = (portName) => {
	if (!portName) return null
	const matchedPort = props.routePorts.find((port) => port?.name === portName)
	if (!matchedPort) return null
	const lat = Number(matchedPort.lat)
	const lon = Number(matchedPort.lon)
	if (!Number.isFinite(lat) || !Number.isFinite(lon)) return null
	return { lat, lon }
}

const buildRouteRenderData = (result) => {
	const keyPoints = normalizeRoutePoints(result)
	const startPortName = result?.start_port || props.selectedStartPoint || ''
	const endPortName = result?.end_port || props.selectedEndPoint || ''

	const startPoint = getPortCoordinateByName(startPortName)
	const endPoint = getPortCoordinateByName(endPortName)

	const linePoints = []
	if (startPoint) {
		linePoints.push(startPoint)
	}
	linePoints.push(...keyPoints)
	if (endPoint) {
		linePoints.push(endPoint)
	}

	const fallbackLinePoints = linePoints.length >= 2 ? linePoints : keyPoints

	return {
		keyPoints,
		linePoints: fallbackLinePoints,
		startPoint,
		endPoint,
		startPortName,
		endPortName
	}
}

const drawRoutePoints = (points, startIndex = 1) => {
	if (!viewer.value) return
	for (const [index, point] of points.entries()) {
		const pointTitle = getRoutePointTitle(startIndex + index)
		const entity = viewer.value.entities.add({
			position: Cesium.Cartesian3.fromDegrees(point.lon, point.lat),
			description: createCoordinateDescription({
				lat: point.lat,
				lon: point.lon,
				title: pointTitle
			}),
			point: {
				pixelSize: 4,
				color: Cesium.Color.WHITE,
				outlineColor: Cesium.Color.BLACK.withAlpha(0.7),
				outlineWidth: 1,
				disableDepthTestDistance: Number.POSITIVE_INFINITY
			},
			label: {
				show: false,
				text: getCoordinateLabelText({ lat: point.lat, lon: point.lon, title: pointTitle }),
				font: '12px sans-serif',
				fillColor: Cesium.Color.WHITE,
				outlineColor: Cesium.Color.BLACK,
				outlineWidth: 2,
				style: Cesium.LabelStyle.FILL_AND_OUTLINE,
				pixelOffset: new Cesium.Cartesian2(0, -18),
				horizontalOrigin: Cesium.HorizontalOrigin.CENTER,
				verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
				disableDepthTestDistance: Number.POSITIVE_INFINITY
			}
		})
		routePointEntities.value.push(entity)
		focusableRouteEntities.value.push(entity)
	}
}

const drawRouteEndpoints = ({ startPoint, endPoint, startName, endName, totalPointCount }) => {
	if (!viewer.value) return

	const buildEndpointEntity = (point, title, color, routeIndex) => {
		if (!point) return
		const entity = viewer.value.entities.add({
			position: Cesium.Cartesian3.fromDegrees(point.lon, point.lat),
			description: createCoordinateDescription({
				lat: point.lat,
				lon: point.lon,
				title
			}),
			point: {
				pixelSize: 18,
				color,
				outlineColor: Cesium.Color.WHITE,
				outlineWidth: 2.4,
				disableDepthTestDistance: Number.POSITIVE_INFINITY
			},
			label: {
				show: false,
				text: getCoordinateLabelText({
					lat: point.lat,
					lon: point.lon,
					title: `${getRoutePointTitle(routeIndex)}${title ? ` (${title})` : ''}`
				}),
				font: '13px sans-serif',
				fillColor: Cesium.Color.WHITE,
				outlineColor: Cesium.Color.BLACK,
				outlineWidth: 2,
				style: Cesium.LabelStyle.FILL_AND_OUTLINE,
				pixelOffset: new Cesium.Cartesian2(0, -22),
				horizontalOrigin: Cesium.HorizontalOrigin.CENTER,
				verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
				disableDepthTestDistance: Number.POSITIVE_INFINITY
			}
		})
		routeEndpointEntities.value.push(entity)
		focusableRouteEntities.value.push(entity)
	}

	buildEndpointEntity(startPoint, startName || 'Start', START_PORT_COLOR, 1)
	buildEndpointEntity(endPoint, endName || 'End', END_PORT_COLOR, totalPointCount)
}

const removeLayerIfExists = (layerRef) => {
	if (!imageryLayers.value || !layerRef.value) return
	try {
		imageryLayers.value.remove(layerRef.value, true)
	} catch (e) {
		// ignore
	}
	layerRef.value = null
}

const createOverlayProvider = async (frame) => {
	return Cesium.SingleTileImageryProvider.fromUrl(frame.cesiumUrl, {
		rectangle: resolveRectangle(frame)
	})
}

const getPlaybackFrames = () => {
	if (props.localRiskFrames.length) return props.localRiskFrames
	if (props.seaIceFrames.length) return props.seaIceFrames
	const resultDates = Array.isArray(props.routeResult?.voyage_dates)
		? props.routeResult.voyage_dates
		: Array.isArray(props.routeResult?.route_dates)
			? props.routeResult.route_dates
			: Array.isArray(props.routeResult?.dates)
				? props.routeResult.dates
				: []

	return resultDates.map((item) => {
		const dateValue = typeof item === 'string' ? item : item?.date || item?.voyage_date || item?.current_date || ''
		const date = dateValue ? new Date(dateValue) : null
		if (!date || Number.isNaN(date.getTime())) {
			return {
				dateKey: '',
				dateLabel: ''
			}
		}
		const year = date.getFullYear()
		const month = String(date.getMonth() + 1).padStart(2, '0')
		const day = String(date.getDate()).padStart(2, '0')
		return {
			dateKey: `${year}-${month}-${day}`,
			dateLabel: `${month}-${day}`
		}
	}).filter((item) => item.dateKey)
}

const renderSeaIceFrame = async (frame) => {
	removeLayerIfExists(seaIceLayer)
	if (!frame?.cesiumUrl || !imageryLayers.value) return
	try {
		const provider = await createOverlayProvider(frame)
		seaIceLayer.value = imageryLayers.value.addImageryProvider(provider)
		seaIceLayer.value.alpha = typeof frame.alpha === 'number' ? frame.alpha : 0.82
	} catch (e) {
		seaIceLayer.value = null
	}
}

const emitPlaybackState = (index, dateKey, dateLabel) => {
	const localRiskFrame = props.localRiskFrames.find((item) => item?.dateKey === dateKey) || null
	currentDateLabel.value = dateLabel || ''
	emit('playback-change', {
		index,
		dateKey: dateKey || '',
		dateLabel: dateLabel || '',
		localRiskFrame
	})
}

const startSynchronizedPlayback = async (routeData) => {
	stopRoutePlayback()

	const routeCartesian = routeData.linePoints.map((point) => Cesium.Cartesian3.fromDegrees(point.lon, point.lat))
	if (routeCartesian.length < 2) {
		clearPlaybackState()
		return
	}

	const timelineFrames = getPlaybackFrames()
	const playbackDates = timelineFrames.map((frame) => frame?.dateKey).filter(Boolean)
	const totalSegments = Math.max(1, routeCartesian.length - 1)
	const uniqueDateCount = Math.max(playbackDates.length, 1)

	if (!playbackDates.length) {
		clearPlaybackState()
		routeLineEntity.value = viewer.value.entities.add({
			polyline: {
				positions: routeCartesian,
				width: 4,
				material: Cesium.Color.fromCssColorString('#ffd166')
			}
		})
		return
	}

	let progress = 2
	let segmentProgress = 0
	let activeDateIndex = 0

	routeLineEntity.value = viewer.value.entities.add({
		polyline: {
			positions: new Cesium.CallbackProperty(() => {
				const maxPointCount = Math.max(2, Math.floor(progress))
				return routeCartesian.slice(0, Math.min(routeCartesian.length, maxPointCount))
			}, false),
			width: 4,
			material: Cesium.Color.fromCssColorString('#ffd166')
		}
	})

	const dateThresholds = Array.from({ length: uniqueDateCount }, (_, index) => {
		if (uniqueDateCount === 1) return totalSegments
		return Math.round((index / (uniqueDateCount - 1)) * totalSegments)
	})

	const applyDateFrame = async (dateIndex) => {
		const safeIndex = Math.min(Math.max(dateIndex, 0), playbackDates.length - 1)
		const dateKey = playbackDates[safeIndex] || ''
		const timelineFrame = timelineFrames.find((item) => item?.dateKey === dateKey) || null
		const dateLabel = timelineFrame?.dateLabel || ''
		const matchedSeaIceFrame = props.seaIceFrames.find((item) => item?.dateKey === dateKey) || null

		emitPlaybackState(safeIndex, dateKey, dateLabel)
		await renderSeaIceFrame(matchedSeaIceFrame)
	}

	await applyDateFrame(activeDateIndex)

	if (totalSegments <= 0) return

	routeAnimationTimer = setInterval(() => {
		if (segmentProgress >= totalSegments) {
			stopRoutePlayback()
			return
		}

		segmentProgress = Math.min(totalSegments, segmentProgress + 1 / ROUTE_PROGRESS_SUBSTEPS)
		progress = Math.min(routeCartesian.length, Math.max(2, 2 + segmentProgress))

		let nextDateIndex = activeDateIndex
		while (
			nextDateIndex + 1 < dateThresholds.length
			&& segmentProgress >= dateThresholds[nextDateIndex + 1]
		) {
			nextDateIndex += 1
		}

		if (nextDateIndex !== activeDateIndex) {
			activeDateIndex = nextDateIndex
			void applyDateFrame(activeDateIndex)
		}
	}, ROUTE_FRAME_INTERVAL)
}

const renderRouteFromResult = async (result) => {
	clearRouteEntities()
	const routeData = buildRouteRenderData(result)
	if (routeData.linePoints.length < 2) {
		clearPlaybackState()
		return
	}

	const routePointStartIndex = routeData.startPoint ? 2 : 1
	drawRoutePoints(routeData.keyPoints, routePointStartIndex)
	drawRouteEndpoints({
		startPoint: routeData.startPoint,
		endPoint: routeData.endPoint,
		startName: routeData.startPortName,
		endName: routeData.endPortName,
		totalPointCount: routeData.linePoints.length
	})
	await startSynchronizedPlayback(routeData)
}

const loadRouteResult = async (result) => {
	if (!viewer.value) initCesium()
	await renderRouteImageLayers(result)
	await renderRouteFromResult(result)
}

const clearRouteResult = () => {
	clearResultLayers()
	removeLayerIfExists(seaIceLayer)
	clearRouteEntities()
	clearPlaybackState()
}

const getBaseLayerProvider = () => {
	return new Cesium.UrlTemplateImageryProvider({
		url: 'https://webst02.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}',
		minimumLevel: 0,
		maximumLevel: 7
	})
}

const initCesium = () => {
	if (!cesiumContainerRef.value) return

	if (!creditEl) {
		creditEl = document.createElement('div')
		creditEl.style.position = 'absolute'
		creditEl.style.width = '0px'
		creditEl.style.height = '0px'
		creditEl.style.overflow = 'hidden'
		creditEl.style.left = '-9999px'
		creditEl.style.top = '-9999px'
		document.body.appendChild(creditEl)
	}

	viewer.value = new Cesium.Viewer(cesiumContainerRef.value, {
		timeline: false,
		animation: false,
		geocoder: false,
		homeButton: false,
		sceneModePicker: false,
		baseLayerPicker: false,
		navigationHelpButton: false,
		fullscreenButton: false,
		baseLayer: new Cesium.ImageryLayer(getBaseLayerProvider()),
		creditContainer: creditEl
	})

	imageryLayers.value = viewer.value.imageryLayers
	setupRouteFocusInteraction()

	try {
		viewer.value.scene.screenSpaceCameraController.minimumZoomDistance = 250000
		viewer.value.scene.screenSpaceCameraController.maximumZoomDistance = 20000000
		viewer.value.camera.flyTo({
			destination: Cesium.Cartesian3.fromDegrees(0.0, 90.0, 8000000.0),
			orientation: {
				heading: 0.0,
				pitch: -Cesium.Math.PI_OVER_TWO,
				roll: 0.0
			},
			duration: 1.5
		})
	} catch (e) {
		// ignore
	}
}

onMounted(async () => {
	initCesium()
	renderPortEntities()

	if (props.routeResult) {
		await loadRouteResult(props.routeResult)
	}
})

watch(
	() => [props.routePorts, props.selectedStartPoint, props.selectedEndPoint],
	() => {
		renderPortEntities()
	},
	{ deep: true }
)

watch(
	() => props.routeResult,
	(newResult) => {
		if (!newResult) {
			clearRouteResult()
			return
		}
		void loadRouteResult(newResult)
	},
	{ deep: true }
)

watch(
	() => [props.seaIceFrames, props.localRiskFrames],
	() => {
		if (!props.routeResult) return
		void loadRouteResult(props.routeResult)
	},
	{ deep: true }
)

onBeforeUnmount(() => {
	if (routeFocusHandler) {
		routeFocusHandler.destroy()
		routeFocusHandler = null
	}

	if (viewer.value && !viewer.value.isDestroyed()) {
		try {
			viewer.value.destroy()
		} catch (e) {
			// ignore
		}
	}

	try {
		if (creditEl && creditEl.parentNode) {
			creditEl.parentNode.removeChild(creditEl)
		}
	} catch (e) {
		// ignore
	}
	creditEl = null
	portEntities.clear()
	clearResultLayers()
	removeLayerIfExists(seaIceLayer)
	clearRouteEntities()
	clearPlaybackState()
})

defineExpose({ loadRouteResult, clearRouteResult })
</script>

<style scoped lang="scss">
.routeSphereContainer {
	width: 100%;
	height: 100%;
	position: relative;
}

.cesiumContainer {
	width: 100%;
	height: 100%;
	border-radius: 10px;
	overflow: hidden;
}

.currentDateBadge {
	position: absolute;
	top: 12px;
	left: 12px;
	padding: 6px 10px;
	border-radius: 999px;
	background: rgba(8, 13, 20, 0.78);
	border: 1px solid rgba(255, 255, 255, 0.16);
	color: #fff;
	font-size: 12px;
	line-height: 1;
	backdrop-filter: blur(6px);
	pointer-events: none;
}
</style>
