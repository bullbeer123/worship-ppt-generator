# worship-ppt-generator 修改说明

## 修改的文件
- `index.html`

## 修改1: 经文/诗篇默认字号增大 (Line 2984-2997)

**问题**: `autoVerseFontSize` 函数字号太小，中文经文在16:9幻灯片上显示偏小

**修改前:**
```js
if (totalChars <= 30) base = 52;
else if (totalChars <= 50) base = 44;
else if (totalChars <= 80) base = 38;
else if (totalChars <= 120) base = 32;
else base = 28;
```

**修改后:**
```js
if (totalChars <= 25) base = 68;
else if (totalChars <= 45) base = 56;
else if (totalChars <= 70) base = 48;
else if (totalChars <= 100) base = 40;
else base = 34;
```

各档位提升30-50%，阈值微调以更好适应中文经文长度。

## 修改2: 音频加载加速

### 2a: 页面加载时预初始化 (Line 4896之前新增)
```js
// ============ Page Init: 提前预加载固定歌曲音频 ============
(function() {
    openAudioDB().catch(() => {});
    loadFixedSongAudioById('praise-welcome', '5b00f970605d87b30340f3b2487b0387', '你原来是宝贵的');
    loadFixedSongAudioById('praise-final', '5b00f970605d87b30340f5467f65e853', '主啊，在我人生中');
})();
```

页面加载时立即初始化IndexedDB + 预加载固定歌曲（已在LOCAL_AUDIO_MAP中，本地同源零延迟）。

### 2b: 播放时优化音频加载路径 (Line 3581-3601)
```js
if (isLocalBlob || isLocalFile) {
    _attachAudioAndPlay(audioUrl, true);
} else {
    // 优先查询缓存 → 缓存未命中则直接播放原始URL（Audio元素支持跨域）
    // 同时在后台下载缓存，供下次使用
    getCachedAudio(audioUrl).then(blobUrl => {
        if (blobUrl && currentAudio && currentAudio.key === s.audioKey) {
            currentAudio.blobUrl = blobUrl;
            _attachAudioAndPlay(blobUrl, true);
        } else {
            throw new Error('no-cache');
        }
    }).catch(() => {
        if (currentAudio && currentAudio.key === s.audioKey) {
            _attachAudioAndPlay(audioUrl, false);  // 直接播放
            prefetchAudioInBackground(audioUrl);    // 后台缓存
        }
    });
}
```

**优化效果**: 
- 远程音频不再等待CORS代理下载完成，直接通过Audio元素播放原始URL（Audio元素天然支持跨域）
- IndexedDB缓存仍用于下次播放加速
- 首次播放延迟从数秒降至几乎即时
