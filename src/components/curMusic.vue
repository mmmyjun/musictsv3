<template>
    <div class="cur-music-container" v-if="currentPlayingObj.id">
        <div class="left" @click="getLrc(currentPlayingObj)">
            <div class="bgImg" v-loading="currentPlayingObj.needLoadDuration" :style="styleImg(currentPlayingObj)"></div>
            <template v-if="!currentPlayingObj.needLoadDuration">
                <el-icon v-show="!currentPlayingObj.hasError" class="upIcon VideoPlay cursorPointer"
                    @click.stop="toPlayAudio" v-if="!currentPlayingObj.isPlaying">
                    <VideoPlay />
                </el-icon>
                <el-icon v-show="!currentPlayingObj.hasError" class="upIcon VideoPause cursorPointer"
                    @click.stop="toPauseAudio" v-else>
                    <VideoPause />
                </el-icon>
            </template>
        </div>
        <div class="center">
            <div class="musicName">
                <div class="long long-cur">{{ currentPlayingObj.loadingLrc ? '正在下载歌词..' : (!currentPlayingObj.lrc ||
                    !currentPlayingObj.lrc.length ? '暂无歌词' : currentPlayingObj.currentLong) }}</div>
                <div class="long-name textHiddenEllipsis">{{ currentPlayingObj.name }} &nbsp;<span class="artist">{{ currentPlayingObj.artist
                }}</span></div>
            </div>
            <div class="time-progress">
                <div class="time-pro-left">
                    <timeProgress :currentTime="currentTime" @emitEnd="endPlay" :totalTime="totalTime"
                        :value="currentTWidth" :cacheWidth="cacheWidth" @change="changeCurTime" />
                    <audio style="height: 0;opacity:0" ref="audioRef" :src="currentPlayingObj.url"
                        @progress="propgressEvent" @loadedmetadata="loadedmetadata" @durationchange="durationchange"
                        preload="auto" @timeupdate="changeAudio" @error="errorPlay" @play="startPlay" @ended="endPlay"
                        @pause="pausePlay" />
                    <musicTime :currentTime="currentTime" :totalTime="totalTime" />
                </div>
                <div class="time-pro-right flexcenter">
                    <volume v-if="!isMobile" class="cur-op-right" v-show="!currentPlayingObj.hasError"
                        @change="chagneVol" />
                    <el-tooltip class="cur-op-right" content="播放列表" placement="top" :hide-after="100">
                        <div class="to-play-list">
                            <svg @click="setPlayedListVisible"
                                class="icon-m MuiSvgIcon-root MuiSvgIcon-fontSizeMedium css-i4bv87-MuiSvgIcon-root"
                                focusable="false" aria-hidden="true" viewBox="0 0 24 24" data-testid="PlaylistPlayIcon">
                                <path d="M3 10h11v2H3zm0-4h11v2H3zm0 8h7v2H3zm13-1v8l6-4z"></path>
                            </svg>
                        </div>
                    </el-tooltip>
                    <modeOfRepeat class="mode-repeat cur-op-right" :repeatMode="repeatMode" @change="toggleRepeatMode" />
                </div>
            </div>
        </div>
    </div>
</template>
<script setup lang="ts">
import { ref, shallowRef, computed } from "vue"
import { ElMessage } from 'element-plus'
import volume from './volume.vue'
import modeOfRepeat from './modeOfRepeat.vue'
import timeProgress from './timeProgress.vue'
import musicTime from './musicTime.vue'

const props = withDefaults(defineProps<{
    modelValue: TypePlaying;
    currentTime: number;
    totalTime: number;
    repeatMode: number;
    cacheWidth: number,
}>(), {
    currentTime: 0,
    totalTime: 0,
    repeatMode: 0,
    cacheWidth: 0,
})
const emit = defineEmits(['getLrc', 'update:modelValue', 'update:currentTime', 'update:totalTime', 'update:repeatMode','update:cacheWidth', 'playNextOne', 'setPlayedListVisible']);

const isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry/i.test(navigator.userAgent)

enum RepeatMode {
    byOrder,
    single,
    random
}
const setPlayedListVisible = (e: any): void => {
    emit('setPlayedListVisible')
}

const currentPlayingObj = computed<TypePlaying>({
    get: () => props.modelValue,
    set: (val) => {
        emit("update:modelValue", val);
    }
});
const currentTime = computed<number>({
    get: () => props.currentTime,
    set: (val) => {
        emit("update:currentTime", val);
    }
})
const totalTime = computed<number>({
    get: () => props.totalTime,
    set: (val) => {
        emit("update:totalTime", val)
    }
})
const repeatMode = computed({
    get: () => props.repeatMode,
    set: (val) => {
        emit("update:repeatMode", val)
    }
})
const cacheWidth = computed({
    get: () => props.cacheWidth,
    set: (val) => {
        emit("update:cacheWidth", val)
    }
})


const toggleRepeatMode = (e: RepeatMode) => {
    repeatMode.value = e
    localStorage.setItem('__repeat_mode', String(e))
}

const getLrc = async (item: TypePlaying) => {
    currentPlayingObj.value.needLoadDuration = true
    emit('getLrc', item)
}


const audioRef = shallowRef<HTMLAudioElement>() // audio对象存储
let localV_ = localStorage.getItem('_volume');
if (audioRef.value as HTMLAudioElement && audioRef.value?.volume) {
    if (!localV_) {
        localStorage.setItem('_volume', '0.4');
    }
    audioRef.value!.volume = Number(JSON.parse(localV_ as string))
}

// 歌词的数据结构
interface TypeOfLrc {
    text: string,
    time: number
}
const changeAudio = (e: Event) => {
    if (audioRef.value && audioRef.value!.currentTime) {
        currentTime.value = audioRef.value!.currentTime
    }
    let idx = -1
    if (audioRef.value && audioRef.value!.currentTime && typeof audioRef.value!.currentTime == 'number' && currentPlayingObj.value.lrc && currentPlayingObj.value.lrc.length) {
        idx = JSON.parse(JSON.stringify(currentPlayingObj.value.lrc)).findIndex((f: TypePlaying) => f.time >= audioRef.value!.currentTime)
    }
    if (idx == -1) {
        return
    };
    currentPlayingObj.value.currentLong = currentPlayingObj.value.lrc[idx > 0 ? idx -1 : 0].text
}

const currentTWidth = computed(() => {
    let pre: number = currentTime.value;
    let suf: number = totalTime.value;
    let last: number = (pre / suf) * 100;
    return last ? last : 0
})
const changeCurTime = (e: number) => {
    let curT = parseInt(String(e * totalTime.value / 100))
    currentTime.value = curT
    audioRef.value && (audioRef.value!.currentTime = curT)
}
const propgressEvent = (e: Event) => {
    const buffered = audioRef.value?.buffered!;
    let bufferedEnd: number;
    try {
        bufferedEnd = buffered.end(buffered.length - 1);
    }
    catch (err) {
        bufferedEnd = 0;
    }
    cacheWidth.value = (bufferedEnd / totalTime.value) * 100
}

const errorPlay = (): void => {
    if (currentPlayingObj.value.errorTime == 1) {
        emit('playNextOne', false)
        ElMessage.error(`"${currentPlayingObj.value.name}"播放错误,已移除`)
    } else {
        currentPlayingObj.value.errorTime = 1
        emit('playNextOne', true)
    }
    currentPlayingObj.value.hasError = true
}
const startPlay = (e?: any): void => {
    currentPlayingObj.value.isPlaying = true
}

const pausePlay = (e?: any): void => {
    currentPlayingObj.value.isPlaying = false
}
const chagneVol = (e: any): void => {
    audioRef.value!.volume = Number(e)
}

const toPlayAudio = (e: Event) => {
    try {
        audioRef.value?.play()
    } catch (e) {
    }
}
const toPauseAudio = (e: Event) => {
    audioRef.value?.pause()
    currentPlayingObj.value.isPlaying = false
}

// duration(总时长)等信息读取到了， 已准备就绪可以开始播放了.但是兼容性没有durationchange好，所以改成后面这个方法去执行相应内容了
const loadedmetadata = (e: Event) => {
    // console.log('loadedmetadata~~')
}

// 整个方法比loadedmetadata兼容性好
const durationchange = (e: Event) => {
    console.log('durationchange~~')
    if (!audioRef.value) return;
    
    currentTime.value = 0
    let dr = audioRef.value.duration || 0;
    totalTime.value = typeof dr == 'number' ? dr : 0
    let localV_ = localStorage.getItem('_volume');
    if (!localV_) {
        localStorage.setItem('_volume', '0.4');
        audioRef.value!.volume = 0.4;
    } else {
        audioRef.value!.volume = Number(JSON.parse(localV_ as string))
    }

    if (!currentPlayingObj.value.firstInit) {
        // currentPlayingObj.value.firstInit = false
        tryToAutoPlay()
    }
    
}
const tryToAutoPlay = async () => {
    try {
        currentPlayingObj.value.isPlaying = true;
        await audioRef.value?.play()
        startPlay()
    }
    catch (err) {
        console.log('auto play failed because of browser security policy. ', err)
        errorPlay()
    }
    currentPlayingObj.value.needLoadDuration = false
}

// 设置3秒后自动播放下一首
const timer = ref(0)
const endPlay = (event?: any): void => {
    clearTimeout(timer.value)
    timer.value = setTimeout(() => {
        emit('playNextOne', false)
    }, 3000);
}

const playState = computed(() => {
    return !currentPlayingObj.value.needLoadDuration && currentPlayingObj.value.isPlaying ? 'running' : 'paused';
})

const styleImg = (obj: TypePlaying) => {
    return {
        backgroundImage: `url(${obj.poster})`,
        animation: '10s linear infinite rotate360 ' + playState.value
    }
}

defineExpose({
    toPlayAudio,
    toPauseAudio,
    tryToAutoPlay
})
</script>
<style>
.cur-music-container {
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
    height: 100px;
    background-color: #111;
    color: #fff;
}

.cur-music-container .left,
.cur-music-container .right {
    width: 88px;
    height: 88px;
    position: relative;
    display: flex;
    justify-content: center;
    align-items: center;
}

.cur-music-container .left {
    border-radius: 50%;
    box-sizing: border-box;
    padding: 5px;
}

@keyframes rotate360 {
    from {
        transform: rotate(0deg);
    }

    to {
        transform: rotate(360deg);
    }
}

.cur-music-container .right {
    rotate: 90deg;
}

.cur-music-container .center {
    width: calc(100% - 96px);
    text-align: left;
}

.cur-music-container .center>div {
    padding-left: 8px;
}

.cur-music-container img {
    width: 100%;
    height: 100%;
}

.cur-music-container .upIcon {
    position: absolute;
    top: 0;
    left: 0;
    color: white;
    width: 100%;
    height: 100%;
    font-size: 48px;
    z-index: 10;
    border-radius: 50%;
}

.cur-music-container .artist {
    color: grey
}

.cur-music-container>div {
    height: 100%;
}

.cur-music-container .upIcon svg {
    stroke-width: 10px;
}

.cur-music-container .time-progress-container {
    margin-top: 16px;
}

.cur-music-container .time-progress {
    display: flex
}

.cur-op-right,
.cur-op-right svg {
    color: #fff;
    fill: #fff;
}

.time-pro-left {
    display: flex;
    flex-grow: 1;
    line-height: 42px;
    height: 42px;
    flex-direction: column;
}

.time-pro-right {

}

.time-pro-right>div {
    width: 40px;
    text-align: center;
}

.artist {
    font-size: 12px;
    color: rgba(0, 0, 0, 0.6);
}

.long-cur {
    overflow: hidden;
    text-overflow: ellipsis;
    width: 96%;
    white-space: nowrap;
    height: 24px;
}

.long-name {
    font-size: 14px;
    height: 26px;
}
</style>