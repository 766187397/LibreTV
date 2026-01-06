# uniapp web-view 视频自动横屏解决方案

## 问题分析

1. **当前实现限制**：播放器仅在原生全屏模式下尝试锁定横屏，Web全屏模式下不执行
2. **Web-View环境问题**：uniapp web-view默认可能锁定竖屏，需要特殊处理
3. **跨端通信缺失**：前端无法直接控制uniapp应用的屏幕方向
4. **设备兼容性**：不同设备对屏幕方向API支持程度不同

## 解决方案

### 1. 修改前端播放器代码

#### 1.1 增强全屏处理逻辑

修改 `handleFullScreen` 函数，**移除`!isWeb`限制**，在所有全屏模式下都尝试横屏锁定：

```javascript
function handleFullScreen(isFullScreen, isWeb) {
    if (isFullScreen) {
        document.addEventListener('mouseout', handleMouseOut);
        
        // 移除 !isWeb 限制，所有全屏模式都尝试横屏
        lockLandscape();
    } else {
        document.removeEventListener('mouseout', handleMouseOut);
        clearTimeout(hideTimer);
        unlockScreenOrientation();
    }
}
```

#### 1.2 添加屏幕方向控制工具函数

```javascript
// 检测设备是否支持屏幕方向 API
function supportsScreenOrientation() {
    return window.screen && window.screen.orientation;
}

// 锁定屏幕方向为横屏
function lockLandscape() {
    if (supportsScreenOrientation()) {
        try {
            window.screen.orientation.lock('landscape')
                .then(() => console.log('屏幕方向已锁定为横屏'))
                .catch(error => {
                    console.error('锁定屏幕方向失败:', error);
                    // 降级方案：尝试不同横屏模式
                    try {
                        window.screen.orientation.lock('landscape-primary');
                    } catch (e) {
                        console.error('降级锁定屏幕方向失败:', e);
                    }
                    // 通知uniapp切换横屏
                    notifyUniapp('lockLandscape');
                });
        } catch (e) {
            console.error('调用屏幕方向 API 失败:', e);
            // 通知uniapp切换横屏
            notifyUniapp('lockLandscape');
        }
    } else {
        // 不支持屏幕方向API，通知uniapp切换横屏
        notifyUniapp('lockLandscape');
    }
}

// 解锁屏幕方向
function unlockScreenOrientation() {
    if (supportsScreenOrientation() && window.screen.orientation.unlock) {
        try {
            window.screen.orientation.unlock();
            console.log('屏幕方向已解锁');
        } catch (e) {
            console.error('解锁屏幕方向失败:', e);
            notifyUniapp('unlockOrientation');
        }
    } else {
        notifyUniapp('unlockOrientation');
    }
}

// 通知uniapp切换屏幕方向
function notifyUniapp(action) {
    try {
        // 使用postMessage与uniapp通信
        if (window.parent && window.parent.postMessage) {
            window.parent.postMessage({
                action: action
            }, '*');
        }
    } catch (e) {
        console.error('通知uniapp失败:', e);
    }
}
```

#### 1.3 增强视频事件处理

在视频开始播放和进入全屏时，都尝试锁定横屏：

```javascript
// 在视频开始播放时尝试横屏
art.on('video:playing', () => {
    // 仅在全屏模式下尝试横屏
    if (art.fullscreen || art.fullscreenWeb) {
        lockLandscape();
    }
});

// 在视频退出全屏时解锁方向
art.on('video:ended', function () {
    videoHasEnded = true;
    clearVideoProgress();
    art.fullscreen = false;
    unlockScreenOrientation();
});
```

### 2. uniapp 端配置

#### 2.1 修改 pages.json 配置

```json
{
  "pages": [
    {
      "path": "pages/player/player",
      "style": {
        "navigationBarTitleText": "视频播放",
        "pageOrientation": "auto" // 允许自动旋转
      }
    }
  ]
}
```

#### 2.2 修改 web-view 组件

```vue
<template>
  <view>
    <web-view 
      ref="webview" 
      :src="url" 
      allow="fullscreen" 
      @message="handleMessage"
    ></web-view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      url: 'https://tv.hushangkun.dpdns.org?code=766187397'
    };
  },
  methods: {
    // 处理来自web-view的消息
    handleMessage(e) {
      const data = e.detail.data[0];
      if (data.action === 'lockLandscape') {
        // 锁定屏幕为横屏
        this.lockLandscape();
      } else if (data.action === 'unlockOrientation') {
        // 解锁屏幕方向
        this.unlockOrientation();
      }
    },
    
    // 锁定横屏
    lockLandscape() {
      // 使用uniapp的plus API锁定横屏
      plus.screen.lockOrientation('landscape-primary');
    },
    
    // 解锁屏幕方向
    unlockOrientation() {
      // 解锁屏幕方向，恢复自动旋转
      plus.screen.unlockOrientation();
    }
  },
  onLoad() {
    // 页面加载时允许自动旋转
    plus.screen.unlockOrientation();
  },
  onUnload() {
    // 页面卸载时恢复竖屏
    plus.screen.lockOrientation('portrait-primary');
  }
};
</script>
```

## 技术要点

1. **跨端通信**：使用 `postMessage` 实现web-view与uniapp的通信
2. **API降级处理**：优先使用浏览器原生API，失败时通知uniapp处理
3. **多场景覆盖**：在所有全屏模式下都尝试横屏锁定
4. **生命周期管理**：正确处理页面加载和卸载时的屏幕方向

## 预期效果

1. 当用户点击视频全屏按钮时，自动锁定为横屏
2. 无需用户手动开启自动旋转屏幕
3. 支持不同设备和浏览器环境
4. 退出全屏时自动恢复屏幕方向

## 实现步骤

1. 修改前端播放器代码，添加屏幕方向控制工具函数
2. 修改全屏处理逻辑，移除`!isWeb`限制
3. 添加与uniapp通信的`notifyUniapp`函数
4. 在视频事件中添加屏幕方向控制
5. 修改uniapp端的pages.json配置
6. 修改web-view组件，添加消息处理逻辑
7. 测试不同设备和浏览器环境

## 注意事项

1. 需要在真机环境下测试，模拟器可能无法正常模拟屏幕旋转
2. 部分浏览器可能需要用户授权屏幕方向控制
3. uniapp的plus API仅在真机环境下可用
4. 不同设备的屏幕方向锁定效果可能有所差异

