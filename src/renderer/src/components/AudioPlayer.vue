<script setup>
import { onMounted, onUnmounted, reactive, ref, watch } from 'vue'
import {
  PlayOnce,
  PlayCycle,
  ShuffleOne,
  SortOne,
  Like,
  VolumeNotice,
  VolumeMute,
  Play,
  PauseOne,
  ArrowCircleLeft,
  ArrowCircleRight,
  MusicList,
  DocSearchTwo
} from '@icon-park/vue-next'
import { PlayerBus } from '../Events'
import { formatTime } from '../utils'
import { loadLyricsText, timeArr } from '../lyrics'
import PlayList from './PlayList.vue'
// 导入结束
// TODO 加入声音的淡入淡出
const volume = ref(true)
const songsLike = ref(false)
const playMode = ref('cycle')
const playStatus = ref(false)
const musicSrc = ref('https://music.163.com/song/media/outer/url?id=447925558.mp3')
// 当前audio的引用
const audioRef = ref(null)
// 当前音频是否可以播放
const canPlay = ref(false)
const lyricsWindowOpen = ref(false)
const currentLyrics = ref('来播放音乐吧~~')
const musicInfo = reactive({
  musicSrc: '',
  picUrl: '',
  musicTitle: '来播放音乐吧~~',
  plaintMusicTitle: '来播放音乐吧~~'
})

const currentPlayList = reactive([])

const totalTimeStr = ref('0:00')
const currentTimeStr = ref('0:00')
let lyricsTimer

const totalTime = ref(0)
// 左边的时间， 在拖拽时成为目标时间，在真正audio 控件更新时，会同步被更新
const currentTime = ref(0)
// 用来同步进度条 其实是 34.6 这样后面有一位小数，来保证进度条的平滑推进
const currentProcessPrecent = ref(0)
// 调整进度是是否松开了鼠标，也代表着是否保持进度条预览和实际进度的同步 同时标志进入了进度调整的状态
const mouseRelease = ref(true)

let mouseDownElement = ''
let playIfCanplayImmediately = false

const musicListFill = ref('#333')
onMounted(() => {
  // 解决 mouseup 时不在元素上，导致不生效的问题，解决办法: 扩大对象范围
  // 新问题: 会导致在document 随便位置单击时出现停顿，根本原因是在设置时间时间不对，在计算时舍去了部分时间
  // 解决办法: 判定 mousedown 是否在元素内被触发, 记录上一次 mousedown 的地方
  window.addEventListener('mouseup', endChangeProcess)
  window.addEventListener('mousedown', recordMouseDownElement)
  // this.$once('hook:beforeDestroy', () => {
  //   window.removeEventListener('mouseup', endChangeProcess)
  // })
  // 更改是否可以播放
  audioRef.value.addEventListener('canplay', () => {
    canPlay.value = true
    if (playIfCanplayImmediately) {
      startPlay()
      playIfCanplayImmediately = false
    }
  })
  audioRef.value.addEventListener('loadedmetadata', () => {
    totalTime.value = audioRef.value.duration
    totalTimeStr.value = formatTime(totalTime.value)
  })

  audioRef.value.addEventListener('ended', playNext)

  // 注册事件
  PlayerBus.on('musicChange', changeMusicInfo)
  //
  window.addEventListener('message', handleLyricsOpen)
})
onUnmounted(() => {
  // 卸载
  window.removeEventListener('mouseup', endChangeProcess)
  window.removeEventListener('mousedown', recordMouseDownElement)
  PlayerBus.off('musicChange', changeMusicInfo)
  window.removeEventListener('windowClose', handleLyricsOpen)
})
const refPlayList = ref()
// 判定单击的时间戳，上一次改变的时间
// 监视当前进度条的进度，完成在进度条改变时对左边时间显示的改变
watch(currentProcessPrecent, () => {
  // 这个进度条是在 mouseup 的时候才会进行 currentProcessPrecent 的同步，
  // 这导致在快速单击的时候，我们在拖拽时候的逻辑不会生效，因为此时 mouseRelease 已经被置为 true，所以我们对单击进行特判
  // 因为单击，在 mouseup 之后， 会有两次快速的变换 currentProcessPrecent
  // 所以对两次更新的时间进行检测即可 所以我们定义一个全局变量表示上一次的时间戳
  // 失败 -->  使用官方@change 方法代替
  if (mouseRelease.value) {
    // let nowTime = parseInt(new Date().getTime() / 1000 + '')
    // // 判定为单击，直接改变原生audio进度
    // if (nowTime - lastTime < 1) {
    //   audioRef.value.currentTime = prePrecent
    // }
    // lastTime = nowTime
    return
  }
  const targetTime = (currentProcessPrecent.value * totalTime.value) / 100
  // 改变预期进度时间显示
  currentTimeStr.value = formatTime(targetTime)
  // 当前currentTime和Str不一致，currentTime 用来进行歌词展示
  currentTime.value = targetTime
})
// 单击改变时的逻辑，分开无法代替
const changeProcess = (precent) => {
  // 拖拽状态下不执行
  if (!mouseRelease.value) return
  // console.log(parseInt((precent * 10 * totalTime.value) / 1000 + ''))
  audioRef.value.currentTime = parseInt((precent * 10 * totalTime.value) / 1000 + '')
}

// 播放下一首
const playNext = () => {
  console.log('end')
  // 查看当前的模式
  startPause()
  // 单曲循环
  if (playMode.value === 'cycle') {
    startPlay()
  }
  //TODO 获取当前的歌单
}

// 开始点击play process bar
const startChangeProcess = () => {
  // 没有松开鼠标， 停止对进度条和当前音频播放时间的控制
  mouseRelease.value = false
  // const targetTime = (currentProcessPrecent.value * totalTime.value) / 100
  // // console.log(targetTime)
  // currentTimeStr.value = formatTime(targetTime)
}
const endChangeProcess = () => {
  // console.log(e.currentTarget)
  // console.log('单击触发了 mouseup')
  // console.log(mouseDownElement)
  if (typeof mouseDownElement !== 'string' || !mouseDownElement.includes('arco-slider')) return
  // 判定元素 mousedown 是否在元素内被触发
  // 通过进度条获得目标时间
  mouseRelease.value = true
  const targetTime = (currentProcessPrecent.value * totalTime.value) / 100
  // console.log(targetTime)
  audioRef.value.currentTime = targetTime
}

// 更新歌词
const updateLyricsLine = () => {
  currentTime.value = audioRef.value.currentTime
  currentLyrics.value = findLyricsInCurrentTime(parseInt(currentTime.value * 1000 + ''))
  if (childWindow) {
    childWindow.postMessage(currentLyrics.value)
  }
}
const handleTimeUpdate = () => {
  updateLyricsLine()
  // console.log(currentLyrics.value)
  // 进度条没有拖动
  if (mouseRelease.value) {
    currentProcessPrecent.value = parseInt('' + (1000 * currentTime.value) / totalTime.value) / 10
    currentTimeStr.value = formatTime(currentTime.value)
  }
}

// let lastLine = ''
const findLyricsInCurrentTime = (time) => {
  for (let i = 0; i < timeArr.length; i++) {
    // if (time < timeArr[0].time) return timeArr[0].line
    if (i === timeArr.length - 1) {
      // lastLine = timeArr[i].line
      return timeArr[i].line
    }
    if (time >= timeArr[i].time && time < timeArr[i + 1].time) {
      // lastLine = timeArr[i].line
      return timeArr[i].line
    }
  }
  return '暂无歌词'
}
const recordMouseDownElement = (e) => {
  mouseDownElement = e.srcElement.className
  // console.log(mouseDownElement)
}

let childWindow

const handleLyricsOpen = () => {
  console.log(123)
  // console.log(lyricsWindowOpen.value)
  if (lyricsWindowOpen.value) {
    // preload js 关闭窗口
    lyricsWindowOpen.value = false
    if (childWindow) childWindow.close()
  } else {
    // 打开窗口
    lyricsWindowOpen.value = true
    // 判断环境

    // if (is.dev && process.env['ELECTRON_RENDERER_URL']) {
    // childWindow = window.open(process.env['ELECTRON_RENDERER_URL'] + '/rendererLyrics/')
    // childWindow = window.open(
    //   ' http://localhost:5173/rendererLyrics/index.html',
    //   '_blank',
    //   'top=200,left=200,frame=false,height=100,width=400'
    // )
    // 本地打包
    window.electronAPI.getPathAndUrl().then((url) => {
      childWindow = window.open(url, '_blank', 'top=200,left=200,frame=false,height=100,width=400')
    })
    // } else {
    //   childWindow = window.open(join('../rendererLyrics/lyricsIndex.html'))
    // }

    // childWindow.document.write('<h1>Hello</h1>')
  }
}
/////////////////////////////////////////

const changePlayMode = (event, targetPlayMode) => {
  playMode.value = targetPlayMode
}
const startPlay = () => {
  // 判断当前是否可以进行播放
  console.log(canPlay.value)
  if (canPlay.value) {
    // 开始播放
    playStatus.value = true
    audioRef.value.play()
  }
}
const startPause = () => {
  playStatus.value = false
  audioRef.value.pause()
}
const changeMusicInfo = (targetMusicInfo) => {
  if (lyricsTimer) lyricsTimer.clear()
  // 静态方法将一个或者多个源对象中所有可枚举的自有属性复制到目标对象，并返回修改后的目标对象。
  // 动态页面直接改变了
  Object.assign(musicInfo, targetMusicInfo)
  // 改变音乐播放
  changeMusicSrc(musicInfo.musicSrc)
  // 进行歌词的处理
  // 载入解析歌词
  loadLyricsText(musicInfo.musicLyrics)
  // 当前的播放列表
  currentPlayList.splice(0, currentPlayList.length)
  Object.assign(currentPlayList, targetMusicInfo.allPages)
  console.log(targetMusicInfo.allPages)
  console.log(currentPlayList)
}
const changeMusicSrc = (targetMusicSrc) => {
  canPlay.value = false
  musicSrc.value = targetMusicSrc
  // 重新加载资源 但是canplay 会在这个 changeMusicSrc 执行完才会触发，导致手动双击切换歌曲失败
  // 因为canplay会先判断 然后才改变
  // 所以需要异步执行 startplay
  // 采取另一种做法， 使用公共变量让出时间片给load即可，使用事件机制
  // TODO 加入加载音频失败判断  定时器实现
  audioRef.value.load()
  // startPlay() 错误做法
  playIfCanplayImmediately = true
}
// true 为静音
const mute = (option = true) => {
  volume.value = !option
  audioRef.value.muted = option
}
const likeSong = () => {
  // 从musicInfo 中获得当前的音乐
  // 加入到喜欢列表, 喜欢列表在initSetting 中进行读取 可以到主进程进行，因为要刷写到磁盘
  // 但还是需要把列表放到vue中
  songsLike.value = true
}
const dislikeSong = () => {
  // 从喜欢列表取消
  songsLike.value = false
}

const showOrNotPlayList = () => {
  if (refPlayList.value.className === '') {
    refPlayList.value.className = 'show'
    musicListFill.value = '#00aeec'
  } else {
    refPlayList.value.className = ''
    musicListFill.value = '#333'
  }
}
const formatVideoPic = (url) => {
  const result = url.replace('//', 'https://')
  console.log(result)
  return result
}
</script>

<template>
  <div class="audio-player">
    <div class="player-cover">
      <div class="cover">
        <img class="cover-img" :src="formatVideoPic(musicInfo.picUrl)" alt="图片加载失败" />
      </div>
    </div>
    <div class="music-info">
      <div class="title only-one-line">{{ musicInfo.plaintMusicTitle }}</div>
      <div class="lyrics only-one-line">{{ currentLyrics }}</div>
    </div>
    <div class="process-bar">
      <div class="process-bar-text">{{ currentTimeStr }}</div>
      <a-slider
        v-model="currentProcessPrecent"
        :min="0"
        :step="0.1"
        :max="100"
        :style="{ width: '70%' }"
        :show-tooltip="false"
        @change="changeProcess"
        @mousedown="startChangeProcess"
        @mouseup="endChangeProcess"
      />
      <div class="process-bar-text">{{ totalTimeStr }}</div>
      <audio ref="audioRef" :src="musicSrc" @timeupdate="handleTimeUpdate" />
    </div>
    <div class="options">
      <volume-notice
        v-if="volume"
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="mute(true)"
      />
      <volume-mute
        v-if="!volume"
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="mute(false)"
      />
      <like
        v-if="songsLike"
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#ff6a6a', '#ff6a6a']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="dislikeSong"
      />
      <like
        v-if="!songsLike"
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="likeSong"
      />
      <span
        class="option-button lyrics-button"
        :class="lyricsWindowOpen ? 'active' : ''"
        @click="handleLyricsOpen"
        >词</span
      >

      <!--      播放模式，弹窗-->
      <a-popover
        class="popover"
        :content-style="{ padding: 0 }"
        trigger="click"
        popup-container=".options"
      >
        <template #title></template>
        <a-button class="play-mode-button">
          <shuffle-one
            v-if="playMode === 'shuffle'"
            theme="two-tone"
            size="24"
            :fill="['#333', '#2F88FF']"
            stroke-linejoin="miter"
            stroke-linecap="square"
          />
          <sort-one
            v-else-if="playMode === 'list'"
            theme="two-tone"
            size="24"
            :fill="['#333', '#2F88FF']"
            stroke-linejoin="miter"
            stroke-linecap="square"
          />
          <play-once
            v-else-if="playMode === 'cycle'"
            theme="two-tone"
            size="24"
            :fill="['#333', '#2F88FF']"
            stroke-linejoin="miter"
            stroke-linecap="square"
          />
          <play-cycle
            v-else-if="playMode === 'sort'"
            theme="two-tone"
            size="24"
            :fill="['#333', '#2F88FF']"
            stroke-linejoin="miter"
            stroke-linecap="square"
          />
        </a-button>
        <template #content>
          <a-list :bordered="false" :hoverable="true">
            <a-list-item @click="changePlayMode($event, 'shuffle')">
              <shuffle-one
                theme="two-tone"
                size="16"
                :fill="['#333', '#2F88FF']"
                stroke-linejoin="miter"
                stroke-linecap="square"
              /><span class="play-mode-text">随机播放</span></a-list-item
            >
            <a-list-item @click="changePlayMode($event, 'list')">
              <sort-one
                theme="two-tone"
                size="16"
                :fill="['#333', '#2F88FF']"
                stroke-linejoin="miter"
                stroke-linecap="square"
              /><span class="play-mode-text">顺序播放</span></a-list-item
            >
            <a-list-item @click="changePlayMode($event, 'cycle')">
              <play-once
                theme="two-tone"
                size="16"
                :fill="['#333', '#2F88FF']"
                stroke-linejoin="miter"
                stroke-linecap="square"
              /><span class="play-mode-text">单曲循环</span></a-list-item
            >
            <a-list-item @click="changePlayMode($event, 'sort')">
              <play-cycle
                theme="two-tone"
                size="16"
                :fill="['#333', '#2F88FF']"
                stroke-linejoin="miter"
                stroke-linecap="square"
              /><span class="play-mode-text">列表循环</span></a-list-item
            >
          </a-list>
        </template>
      </a-popover>
      <!--下一首-->
      <arrow-circle-left
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
      />
      <!--      !playStatus 默认为true 当前没有在播放，-->
      <play
        v-if="!playStatus"
        class="option-button"
        theme="two-tone"
        size="40"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="startPlay"
      />
      <pause-one
        v-if="playStatus"
        class="option-button"
        theme="two-tone"
        size="40"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="startPause"
      />
      <arrow-circle-right
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="['#333', '']"
        stroke-linejoin="miter"
        stroke-linecap="square"
      />
      <!--      播放列表-->
      <music-list
        class="option-button"
        theme="two-tone"
        size="24"
        :fill="musicListFill"
        stroke-linejoin="miter"
        stroke-linecap="square"
        @click="showOrNotPlayList"
      />
      <!--      TODO 实现播放列表-->
      <play-list ref="refPlayList" :height="795" :width="400" :data="currentPlayList" />
      <!--      TODO 歌词选择-->
      <doc-search-two
        theme="two-tone"
        size="24"
        :fill="['#d0021b', '#7ed321']"
        stroke-linejoin="miter"
        stroke-linecap="square"
      />
      <!--      TODO 进行歌词进度匹配-->
    </div>
  </div>
</template>

<style scoped lang="scss">
.audio-player {
  box-sizing: content-box;
  border-top: 1px solid rgb(222, 222, 222);
  background-color: rgb(246, 246, 246);
  width: 100%;
  height: 80px;
  .player-cover {
    float: left;
    width: 150px;
    height: 80px;
    padding: 10px 0 0 20px;
    .cover {
      width: 120px;
      height: 70px;
      .cover-img {
        width: 120px;
        height: 70px;
      }
    }
  }
  .music-info {
    float: left;
    width: 20%;
    //background: #ffffff;
    padding: 16px 0 0 0;
    text-align: left;

    .only-one-line {
      // 文字省略
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 1;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    .title {
      font-weight: 600;
      color: gray;
    }
    .lyrics {
      text-align: left;
      padding: 10px 0 0 0;
      font-size: 18px;
      font-weight: 700;
    }
  }
  .process-bar {
    float: left;
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
    align-content: center;
    width: 40%;
    height: 80px;
    //background: red;
    .process-bar-text {
      height: fit-content;
      padding: 0 10px;
    }
    .arco-slider {
      :deep(.arco-slider-track) {
        .arco-slider-btn::after {
          display: none;
        }
      }
      :deep(.arco-slider-track:hover) {
        .arco-slider-btn::after {
          display: block;
        }
      }
    }
  }
  .options {
    position: relative;
    float: right;
    display: flex;
    flex-direction: row;
    justify-content: space-around;
    align-items: center;
    width: 26%;
    height: 80px;
    //background: #5c5e66;
    .option-button {
      cursor: pointer;
    }
    .play-mode-button {
      background: none;
      padding: 0;
    }
    .play-mode-text {
      font-size: 12px;
      line-height: 16px;
      padding-left: 5px;
    }
    :deep(.arco-list-content) {
      cursor: pointer;
    }
    :deep(.arco-list-item-content) {
      display: flex;
      align-items: center;
    }
    .play-list {
      position: absolute;
      right: -400px;
      bottom: 85px;
    }
    .play-list.show {
      animation: show 0.3s ease-out infinite;
      animation-iteration-count: 1;
      animation-fill-mode: forwards; /*让动画停留在最后一帧 */
    }
    @keyframes show {
      0% {
        right: -400px;
        opacity: 0.3;
      }
      100% {
        right: 0;
        opacity: 1;
      }
    }
  }
  .lyrics-button {
    font-size: 18px;
    font-weight: 500;
    font-family: 'Microsoft YaHei UI', ui-sans-serif;
  }
  .lyrics-button.active {
    color: #00aeec;
  }
}
</style>
