<template>
	<div :style="{ display: 'flex', 'flex-direction': 'column', gap: '5px' }">
		<div :style="{ display: 'flex', gap: '5px','flex-direction': 'column', }">
			<div>
				摄像头：<b>{{ flvMediaInfo.anchorName }}</b>
			</div>
			<div>
				在线：
				<span v-if="flvMediaInfo.onlineStatus" :style="{ color: 'green' }">是</span>
				<span v-if="!flvMediaInfo.onlineStatus" :style="{ color: 'red' }">否</span>
				<!-- <checbox size="small" :checked="flvMediaInfo.hasAudio" disabled="true">
					hasAudio
				</checbox> -->
			</div>
			<div>
				<input type="checkbox" :checked="flvMediaInfo.hasAudio" disabled="true">hasAudio</input>
			</div>
		</div>
		<div :style="{ display: 'flex', gap: '5px' }">
			<button type="primary" size="small" @click="handlePlay" :disabled="!flvMediaInfo.onlineStatus">
				播放
			</button>
			<button type="primary" size="small" @click="handleStop">
				停止
			</button>
		</div>
		<div>
			<video :id="flvMediaInfo.anchorName" :src="mediaInfoUrl" @error="videoErrorCallback" :danmu-list="danmuList"
				enable-danmu danmu-btn controls></video>
		</div>
	</div>
</template>

<script setup lang="ts">
	import { ref, onMounted, onBeforeUnmount, watch } from "vue";
	import Env from "../../conf/env";
	const serverUrl = Env.directServerUrl;

	const mediaInfoUrl = ref();
	const canStop = ref<boolean>(false);
	const videoRef = ref(null);
	const flvMediaInfo = ref<FlvMediaInfo>({
		hasAudio: false,
		onlineStatus: false,
		anchorName: "--",
	});
	const fgOnlineStatusRef = ref<boolean>(flvMediaInfo.value.onlineStatus);
	const checkOnlineStatusInterval = ref();
	const interval = ref();

	const danmuList = ref([{
		text: '第 1s 测试弹幕',
		color: '#ff0000',
		time: 1
	},
	{
		text: '第 3s 测试弹幕',
		color: '#ff00ff',
		time: 3
	}
	]);

	const videoErrorCallback = (e : any) => {
		uni.showModal({
			content: e.target.errMsg,
			showCancel: false
		})
	}

	const { getLiveInfo } = defineProps<{
		getLiveInfo : () => TLiveInfo | undefined;
	}>();

	type FlvMediaInfo = {
		hasAudio : boolean;
		onlineStatus : boolean;
		anchorName : string;
	};

	export type TLiveInfo = {
		method : "permanent" | "temp";
		code : string;
		playAuthCode : string;
	};

	watch(flvMediaInfo, () => {
		fgOnlineStatusRef.value = flvMediaInfo.value.onlineStatus;
	});

	onBeforeUnmount(() => {
		return () => {
			clearInterval(checkOnlineStatusInterval.value);
			clearInterval(interval.value);
			if (videoRef.value) {
				videoRef.value.stop();
			}
		};
	});

	const handlePlay = () => {
		const liveInfo = getLiveInfo();
		if (!liveInfo) {
			return;
		}
		const mediaInfoGetUrl = `${serverUrl}/live/getMediaInfo/${liveInfo.method}/${liveInfo.code}/${liveInfo.playAuthCode}.flv`;
		uni.request({
			url: mediaInfoGetUrl,
			method: 'GET',
			data: {
			},
			header: {
			},
			success: (res) => {
				const mediaInfo : FlvMediaInfo = res.data.data;
				flvMediaInfo.value = mediaInfo;
				if (!mediaInfo.onlineStatus) {
					uni.showModal({
						content: `anchor: ${mediaInfo.anchorName} not on the air`,
						showCancel: false
					})
					return;
				}
				videoRef.value.play();
			}
		});
	};

	const handleStop = () => {
		if (videoRef.value) {
			console.log(videoRef.value.stop)
			videoRef.value.stop();
		}
	};

	const handleViedoClick = (e : any) => {
		handlePlay();
	};

	onMounted(() => {
		const liveInfo = getLiveInfo();
		if (!liveInfo) {
			return;
		}
		let videoUrl = `${serverUrl}/live/${liveInfo.method}/${liveInfo.code}/${liveInfo.playAuthCode}.flv`;
		mediaInfoUrl.value = videoUrl;
		videoRef.value = uni.createVideoContext(flvMediaInfo.value.anchorName);

		const mediaInfoGetUrl = `${serverUrl}/live/getMediaInfo/${liveInfo.method}/${liveInfo.code}/${liveInfo.playAuthCode}.flv`;
		uni.request({
			url: mediaInfoGetUrl,
			method: 'GET',
			data: {
			},
			header: {
			},
			success: (res) => {
				const mediaInfo : FlvMediaInfo = res.data.data;
				flvMediaInfo.value = mediaInfo;
				if (!mediaInfo.onlineStatus) {
					uni.showModal({
						content: `anchor: ${mediaInfo.anchorName} not on the air`,
						showCancel: false
					})
					return;
				}
			}
		});

		checkOnlineStatusInterval.value = setInterval(() => {
			if (fgOnlineStatusRef.value === true) {
				return;
			}
			const liveInfo = getLiveInfo();
			if (!liveInfo) {
				return;
			}
			const mediaInfoGetUrl = `${serverUrl}/live/getMediaInfo/${liveInfo.method}/${liveInfo.code}/${liveInfo.playAuthCode}.flv`;
			uni.request({
				url: mediaInfoGetUrl,
				method: 'GET',
				data: {
				},
				header: {
				},
				success: (res) => {
					console.log(res.data);
					const mediaInfo : FlvMediaInfo = res.data.data;
					flvMediaInfo.value = mediaInfo;
					if (!mediaInfo.onlineStatus) {
						uni.showModal({
							content: `anchor: ${mediaInfo.anchorName} not on the air`,
							showCancel: false
						})
						return;
					}
				}
			});
		}, 60000);
		checkOnlineStatusInterval.value;

		interval.value = setInterval(() => {
			if (videoRef.value) {
				canStop.value = true;
				const buffered = videoRef.value.buffered;
				if (videoRef.value.paused && buffered && buffered.length > 0) {
					const maxBufferedSec = 3 * 60;
					const currentTime = videoRef.value.currentTime;
					if (buffered.end(0) - currentTime > maxBufferedSec) {
						console.log("vedio paused, buffered max, unload the media data");
						videoRef.value.stop();
					}
				}
			} else {
				canStop.value = false;
			}
		}, 1000);
	});
</script>

<style></style>