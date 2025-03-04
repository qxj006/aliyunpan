<script setup lang='ts'>
import { useAppStore } from '../store'
import { onBeforeUnmount, onMounted } from 'vue'
import Artplayer from 'artplayer'
import FlvJs from 'flv.js'
import HlsJs from 'hls.js'
import AliFile from '../aliapi/file'
import AliDirFileList from '../aliapi/dirfilelist'
import levenshtein from 'fast-levenshtein'
import { type SettingOption } from 'artplayer/types/setting'
import { type Option } from 'artplayer/types/option'

const appStore = useAppStore()
const pageVideo = appStore.pageVideo!
let autoPlayNumber = 0
let ArtPlayerRef: Artplayer

const options: Option = {
  id: 'artPlayer',
  container: '#artPlayer',
  url: '',
  volume: 1,
  autoSize: false,
  autoMini: true,
  loop: false,
  flip: true,
  playbackRate: true,
  aspectRatio: true,
  setting: true,
  hotkey: true,
  pip: true,
  airplay: true,
  mutex: true,
  fullscreen: true,
  fullscreenWeb: true,
  subtitleOffset: false,
  screenshot: true,
  miniProgressBar: false,
  playsInline: true,
  moreVideoAttr: {
    // @ts-ignore
    'webkit-playsinline': true,
    playsInline: true
  },
  customType: {
    flv: (video: HTMLMediaElement, url: string) => playFlv(video, url, ArtPlayerRef),
    m3u8: (video: HTMLMediaElement, url: string) => playM3U8(video, url, ArtPlayerRef)
  }
}

const playM3U8 = (video: HTMLMediaElement, url: string, art: Artplayer) => {
  if (HlsJs.isSupported()) {
    // @ts-ignore
    if (art.hls) art.hls.destroy()
    const hls = new HlsJs({
      maxBufferLength: 20,
      maxBufferSize: 60 * 1000 * 1000
    })
    hls.detachMedia()
    hls.loadSource(url)
    hls.attachMedia(video)
    hls.on(HlsJs.Events.MANIFEST_PARSED, async () => {
      await art.play()
      await getVideoCursor(art, pageVideo.play_cursor)
    })
    hls.on(HlsJs.Events.ERROR, (event, data) => {
      const errorType = data.type
      const errorDetails = data.details
      const errorFatal = data.fatal
      if (errorFatal) { // 尝试修复致命错误
        if (errorType === HlsJs.ErrorTypes.MEDIA_ERROR) {
          hls.recoverMediaError()
        } else if (errorType === HlsJs.ErrorTypes.NETWORK_ERROR) {
          art.emit('video:ended', data)
        } else {
          hls.destroy()
        }
      }
    })
    // @ts-ignore
    art.hls = hls
    art.on('destroy', () => hls.destroy())
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    video.src = url
  } else {
    art.notice.show = 'Unsupported playback format: m3u8'
  }
}

const playFlv = (video: HTMLMediaElement, url: string, art: Artplayer) => {
  if (FlvJs.isSupported()) {
    // @ts-ignore
    if (art.flv) art.flv.destroy()
    const flv = FlvJs.createPlayer(
      { url: url, type: 'flv', withCredentials: true, cors: true },
      { referrerPolicy: 'same-origin' }
    )
    flv.attachMediaElement(video)
    flv.load()
    // @ts-ignore
    art.flv = flv
    art.on('destroy', () => flv.destroy())
  } else {
    art.notice.show = 'Unsupported playback format: flv'
  }
}

type selectorItem = {
  url: string;
  html: string;
  name?: string;
  default?: boolean;
  file_id?: string;
  play_cursor?: number;
}

onMounted(async () => {
  const name = pageVideo.file_name || '视频在线预览'
  setTimeout(() => {
    document.title = name
  }, 1000)
  // 创建播放窗口
  await createVideo(name)
  await defaultSetting(ArtPlayerRef)
  // 获取视频信息
  await getPlayList(ArtPlayerRef)
  await getVideoInfo(ArtPlayerRef)
})

const createVideo = async (name: string) => {
  // 初始化
  ArtPlayerRef = new Artplayer(options)
  ArtPlayerRef.title = name
  // 自定义热键
  // enter
  ArtPlayerRef.hotkey.add(13, () => {
    ArtPlayerRef.fullscreen = !ArtPlayerRef.fullscreen
  })
  // z
  ArtPlayerRef.hotkey.add(90, () => {
    ArtPlayerRef.playbackRate = 1
  })
  // x
  ArtPlayerRef.hotkey.add(88, () => {
    ArtPlayerRef.playbackRate -= 0.5
  })
  // c
  ArtPlayerRef.hotkey.add(67, () => {
    ArtPlayerRef.playbackRate += 0.5
  })
  // 获取用户配置
  const storage = ArtPlayerRef.storage
  if (storage.get('playListMode') === undefined) storage.set('playListMode', true)
  if (storage.get('autoJumpCursor') === undefined) storage.set('autoJumpCursor', true)
  if (storage.get('subTitleListMode') === undefined) storage.set('subTitleListMode', false)
  const volume = storage.get('videoVolume')
  if (volume) ArtPlayerRef.volume = parseFloat(volume)
  const muted = storage.get('videoMuted')
  if (muted) ArtPlayerRef.muted = muted === 'true'
  // 监听事件
  ArtPlayerRef.on('ready', async () => {
    // @ts-ignore
    if (!ArtPlayerRef.hls && !ArtPlayerRef.flv) {
      await ArtPlayerRef.play()
      await getVideoCursor(ArtPlayerRef, pageVideo.play_cursor)
    }
    // 视频播放完毕
    ArtPlayerRef.on('video:ended', () => {
      updateVideoTime()
      if (storage.get('autoPlayNext')) {
        const autoPlayNext = () => {
          const item = playList[++autoPlayNumber]
          if (autoPlayNumber >= playList.length) {
            ArtPlayerRef.notice.show = '视频播放完毕'
            return
          }
          if (item.file_id !== pageVideo.file_id) {
            refreshSetting(ArtPlayerRef, item)
            getPlayList(ArtPlayerRef, item.file_id)
          } else {
            autoPlayNext()
          }
        }
        autoPlayNext()
      }
    })
    // 播放已暂停
    ArtPlayerRef.on('video:pause', () => {
      updateVideoTime()
    })
    // 音量发生变化
    ArtPlayerRef.on('video:volumechange', () => {
      storage.set('videoVolume', ArtPlayerRef.volume)
      storage.set('videoMuted', ArtPlayerRef.muted ? 'true' : 'false')
    })
  })
}

const curDirFileList: any[] = []
const childDirFileList: any[] = []
const getDirFileList = async (dir_id: string, hasDir: boolean, category: string = '', filter?: RegExp): Promise<any[]> => {
  if (curDirFileList.length === 0 || (hasDir && childDirFileList.length === 0)) {
    const dir = await AliDirFileList.ApiDirFileList(pageVideo.user_id, pageVideo.drive_id, dir_id, '', 'name asc', '')
    if (!dir.next_marker) {
      for (let item of dir.items) {
        const fileInfo = {
          html: item.name,
          category: item.category,
          name: item.name,
          file_id: item.file_id,
          ext: item.ext,
          isDir: item.isDir
        }
        if (!hasDir) curDirFileList.push(fileInfo)
        else childDirFileList.push(fileInfo)
      }
    }
  }
  const filterList = hasDir ? [...childDirFileList, ...curDirFileList].sort((a, b)=> a.name.localeCompare(b.name, 'zh-CN')): curDirFileList
  if (category) {
    return filterList.filter(file => file.category === category)
  }
  if (filter) {
    return filterList.filter(file => filter.test(file.ext))
  }
  return filterList
}


const refreshSetting = async (art: Artplayer, item: any) => {
  // 刷新文件
  pageVideo.html = item.html
  pageVideo.play_cursor = item.play_cursor
  pageVideo.file_name = item.html
  pageVideo.file_id = item.file_id || ''
  // 刷新信息
  await getVideoInfo(art)
}

const defaultSetting = async (art: Artplayer) => {
  art.setting.add({
    name: 'autoJumpCursor',
    width: 250,
    html: '自动跳转',
    tooltip: art.storage.get('autoJumpCursor') ? '跳转到历史进度' : '关闭',
    switch: art.storage.get('autoJumpCursor'),
    onSwitch: async (item: SettingOption) => {
      item.tooltip = item.switch ? '关闭' : '跳转到历史进度'
      art.storage.set('autoJumpCursor', !item.switch)
      return !item.switch
    }
  })
  art.setting.add({
    name: 'autoPlayNext',
    width: 250,
    html: '自动连播',
    tooltip: art.storage.get('autoPlayNext') ? '开启' : '关闭',
    switch: art.storage.get('autoPlayNext'),
    onSwitch: (item: SettingOption) => {
      item.tooltip = item.switch ? '关闭' : '开启'
      art.notice.show = '自动连播' + item.tooltip
      art.storage.set('autoPlayNext', !item.switch)
      return !item.switch
    }
  })
  art.setting.add({
    name: 'playListMode',
    width: 250,
    html: '列表模式',
    tooltip: art.storage.get('playListMode') ? '同文件夹' : '同专辑',
    switch: art.storage.get('playListMode'),
    onSwitch: async (item: SettingOption) => {
      item.tooltip = item.switch ? '同专辑' : '同文件夹'
      art.storage.set('playListMode', !item.switch)
      await getPlayList(art)
      return !item.switch
    }
  })
}

const getVideoInfo = async (art: Artplayer) => {
  // 获取视频链接
  const data: any = await AliFile.ApiVideoPreviewUrl(pageVideo.user_id, pageVideo.drive_id, pageVideo.file_id)
  if (data) {
    // 画质
    const qualitySelector: selectorItem[] = []
    if (data.urlQHD) qualitySelector.push({ url: data.urlQHD, html: '原画' })
    if (data.urlFHD) qualitySelector.push({ url: data.urlFHD, html: '全高清 1080P' })
    if (data.urlHD) qualitySelector.push({ url: data.urlHD, html: '高清 720P' })
    if (data.urlSD) qualitySelector.push({ url: data.urlSD, html: '标清 540P' })
    if (data.urlLD) qualitySelector.push({ url: data.urlLD, html: '流畅 480P' })
    const qualityDefault = qualitySelector.find((item) => item.default) || qualitySelector[0]
    qualityDefault.default = true
    art.url = qualityDefault.url
    art.controls.update({
      name: 'quality',
      index: 20,
      position: 'right',
      style: { marginRight: '10px' },
      html: qualityDefault ? qualityDefault.html : '',
      selector: qualitySelector,
      onSelect: async (item: selectorItem) => {
        await art.switchQuality(item.url)
      }
    })
    // 内嵌字幕
    const subtitles = data.subtitles || []
    if (subtitles.length > 0) {
      for (let i = 0; i < subtitles.length; i++) {
        embedSubSelector.push({
          html: '内嵌:  ' + subtitles[i].language,
          name: subtitles[i].language,
          url: subtitles[i].url,
          default: i === 0
        })
      }
      art.subtitle.url = embedSubSelector[0].url
      let subtitleSize = art.storage.get('subtitleSize') || '30px'
      art.subtitle.style('fontSize', subtitleSize)
    }
    // 字幕列表
    await getSubTitleList(art)
  }
}

let playList: selectorItem[] = []
const getPlayList = async (art: Artplayer, file_id?: string) => {
  if (!file_id) {
    let fileList: any
    if (!art.storage.get('playListMode')) {
      fileList = await AliFile.ApiListByFileInfo(pageVideo.user_id, pageVideo.drive_id, pageVideo.file_id, 100)
    } else {
      fileList = await getDirFileList(pageVideo.parent_file_id, false, 'video') || []
    }
    if (fileList && fileList.length > 1) {
      playList = []
      for (let i = 0; i < fileList.length; i++) {
        playList.push({
          url: fileList[i].url,
          html: fileList[i].html,
          name: fileList[i].name,
          file_id: fileList[i].file_id,
          play_cursor: fileList[i].play_cursor,
          default: fileList[i].file_id === pageVideo.file_id
        })
      }
    }
  } else {
    for (let list of playList) {
      if (list.file_id === file_id) {
        list.default = true
        break
      }
    }
  }
  if (playList.length > 1) {
    art.controls.update({
      name: 'playList',
      index: 10,
      position: 'right',
      style: { padding: '0 10px' },
      html: pageVideo.html.length > 20 ? pageVideo.html.substring(0, 40) + '...' : pageVideo.html,
      selector: playList,
      onSelect: async (item: SettingOption) => {
        updateVideoTime()
        await refreshSetting(art, item)
        return item.html.length > 20 ? item.html.substring(0, 40) + '...' : item.html
      }
    })
  }
}

const getVideoCursor = async (art: Artplayer, play_cursor?: number) => {
  if (art.storage.get('autoJumpCursor')) {
    // 进度
    if (play_cursor) {
      art.currentTime = play_cursor
    } else {
      const info = await AliFile.ApiFileInfo(pageVideo.user_id, pageVideo.drive_id, pageVideo.file_id)
      if (info?.play_cursor) {
        art.currentTime = info?.play_cursor
      } else if (info?.user_meta) {
        const meta = JSON.parse(info?.user_meta)
        if (meta.play_cursor) {
          art.currentTime = parseFloat(meta.play_cursor)
        }
      }
    }
  }
}

const loadOnlineSub = async (art: Artplayer, item: any) => {
  const data = await AliFile.ApiFileDownloadUrl(pageVideo.user_id, pageVideo.drive_id, item.file_id, 14400)
  if (typeof data !== 'string' && data.url && data.url != '') {
    await art.subtitle.switch(data.url, {
      name: item.name,
      type: item.ext,
      encoding: 'utf-8',
      escape: true
    })
    return item.html
  } else {
    art.notice.show = `加载${item.name}字幕失败`
  }
}

// 内嵌字幕
const embedSubSelector: selectorItem[] = []
const getSubTitleList = async (art: Artplayer) => {
  // 尝试加载当前文件夹字幕文件
  let subSelector: selectorItem[]
  const hasDir = art.storage.get('subTitleListMode')
  // 加载二级目录(仅加载一个文件夹)
  let file_id = ''
  if (hasDir) {
    try {
      file_id = curDirFileList.find(file => file.isDir).file_id
    } catch (err) {
    }
  } else {
    file_id = pageVideo.parent_file_id
  }
  let onlineSubSelector = await getDirFileList(file_id, hasDir, '', /srt|vtt|ass/) || []
  // console.log('onlineSubSelector', onlineSubSelector)
  subSelector = [...embedSubSelector, ...onlineSubSelector]
  if (subSelector.length === 0) {
    subSelector.push({ html: '无可用字幕', name: '', url: '', default: true })
  }
  if (embedSubSelector.length === 0 && onlineSubSelector.length > 0) {
    const similarity = { distance: 999, index: 0 }
    for (let i = 0; i < subSelector.length; i++) {
      // 莱文斯坦距离算法(计算相似度)
      const distance = levenshtein.get(pageVideo.file_name, subSelector[i].html, { useCollator: true })
      if (similarity.distance > distance) {
        similarity.distance = distance
        similarity.index = i
      }
    }
    // 自动加载同名字幕
    if (similarity.distance !== 999) {
      let selectorItem = subSelector[similarity.index]
      let subtitleSize = art.storage.get('subtitleSize') || '30px'
      art.subtitle.style('fontSize', subtitleSize)
      subSelector.forEach(v => v.default = false)
      selectorItem.default = true
      await loadOnlineSub(art, selectorItem)
    }
  }
  const subDefault = subSelector.find((item) => item.default) || subSelector[0]
  // 字幕设置面板
  art.setting.update({
    name: 'Subtitle',
    width: 250,
    html: '字幕设置',
    tooltip: art.subtitle.show ? (subDefault.url !== '' ? '字幕开启' : subDefault.html) : '字幕关闭',
    selector: [
      {
        html: '字幕开关',
        tooltip: subDefault.url !== '' ? '开启' : '关闭',
        switch: subDefault.url !== '',
        onSwitch: (item: SettingOption) => {
          if (subDefault.url !== '') {
            item.tooltip = item.switch ? '关闭' : '开启'
            art.subtitle.show = !item.switch
            art.notice.show = '字幕' + item.tooltip
            let currentItem = Artplayer.utils.queryAll('.art-setting-panel.art-current .art-setting-item:nth-of-type(n+3)')
            if (currentItem.length > 0) {
              currentItem.forEach((current: HTMLElement) => {
                if (item.switch) {
                  !art.subtitle.url && Artplayer.utils.removeClass(current, 'art-current')
                  Artplayer.utils.addClass(current, 'disable')
                  item.$parentItem.tooltip = subDefault.url !== '' ? '字幕开启' : subDefault.html
                } else {
                  item.$parentItem.tooltip = '字幕开启'
                  Artplayer.utils.removeClass(current, 'disable')
                }
              })
            }
            return !item.switch
          } else {
            return false
          }
        }
      },
      {
        html: '字幕列表',
        tooltip: art.storage.get('subTitleListMode') ? '含子文件夹' : '同文件夹',
        switch: art.storage.get('subTitleListMode'),
        onSwitch: async (item: SettingOption) => {
          item.tooltip = item.switch ? '同文件夹' : '含子文件夹'
          art.storage.set('subTitleListMode', !item.switch)
          await getSubTitleList(art)
          return !item.switch
        }
      },
      {
        html: '字幕偏移',
        tooltip: '0s',
        range: [0, -5, 5, 0.1],
        onChange(item: SettingOption) {
          art.subtitleOffset = item.range
          return item.range + 's'
        }
      },
      {
        html: '字幕大小',
        tooltip: '30px',
        range: [30, 20, 50, 5],
        onChange: (item: SettingOption) => {
          let size = item.range + 'px'
          art.storage.set('subtitleSize', size)
          art.subtitle.style('fontSize', size)
          return size
        }
      },
      ...subSelector
    ],
    onSelect: async (item: SettingOption, element: HTMLDivElement) => {
      if (art.subtitle.show) {
        if (!item.file_id) {
          art.notice.show = ''
          art.subtitle.switch(item.url, {
            name: item.name,
            encoding: 'utf-8',
            escape: true
          })
          return item.html
        } else {
          return await loadOnlineSub(art, item)
        }
      } else {
        art.notice.show = '未开启字幕'
        Artplayer.utils.removeClass(element, 'art-current')
        return false
      }
    }
  })
}

const updateVideoTime = () => {
  AliFile.ApiUpdateVideoTime(
    pageVideo.user_id,
    pageVideo.drive_id,
    pageVideo.file_id,
    ArtPlayerRef.currentTime
  )
}
const handleHideClick = () => {
  updateVideoTime()
  window.close()
}

onBeforeUnmount(() => {
  ArtPlayerRef && ArtPlayerRef.destroy(false)
})

</script>

<template>
  <a-layout style='height: 100vh' draggable='false'>
    <a-layout-header id='xbyhead' draggable='false'>
      <div id='xbyhead2' class='q-electron-drag'>
        <a-button type='text' tabindex='-1'>
          <i class='iconfont iconfile_video'></i>
        </a-button>
        <div class='title'>{{ appStore.pageVideo?.file_name || '视频在线预览' }}</div>
        <div class='flexauto'></div>
        <a-button type='text' tabindex='-1' @click='handleHideClick()'>
          <i class='iconfont iconclose'></i>
        </a-button>
      </div>
    </a-layout-header>
    <a-layout-content style='height: calc(100vh - 42px)'>
      <div id='artPlayer' style='width: 100%; height: 100%;text-overflow: ellipsis;white-space: nowrap;'></div>
    </a-layout-content>
  </a-layout>
</template>

<style>
.disable {
  cursor: not-allowed;
  pointer-events: none;
  background-color: transparent;
  color: #ACA899;
}
</style>
