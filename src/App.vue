<script>
import { ref, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const msg = ref('');
    const nickName = ref('');
    const showProfileModal = ref(false);
    const maxCharacters = ref(80);
    const keyboardHeight = ref(0);
    const danmuList = ref([]);
    const maxDanmuCount = ref(19);
    // const maxContainerHeight = ref(window.innerHeight - 100);
    const maxContainerHeight = window.innerHeight;
    const showLimitWarning = ref(false);

    let socket;
    let clientId = ref(''); // 为当前客户端创建一个唯一 ID
    onMounted(() => {
      generateClientId();
      initializeWebSocket();
      checkAvatarAndNickname();
    });

    let unmounted = false;
    onBeforeUnmount(() => {
      unmounted = true;
      if (socket) {
        socket.close();
      }
    });
    

    function generateClientId() {
      const storedId = localStorage.getItem('clientId');
      if (storedId) {
        clientId.value = storedId;
      } else {
        const newId = 'client_' + Date.now() + '_' + Math.floor(Math.random() * 10000);
        clientId.value = newId;
        localStorage.setItem('clientId', newId);
      }
    }


    function initializeWebSocket() {
      // 后端地址：构建时可用 VITE_WS_URL 指定；默认连当前网站的 /ws，
      // 由 nginx（线上）或 vite 开发服务器（本地）转发到后端的 8080 端口
      const wsUrl = import.meta.env.VITE_WS_URL
        || `${location.protocol === 'https:' ? 'wss' : 'ws'}://${location.host}/ws`;
      socket = new WebSocket(wsUrl);
      socket.onopen = () => console.log('WebSocket 连接成功');
      socket.onmessage = (event) => {
        try {
          handleDanmuChange(JSON.parse(event.data));
        } catch (e) {
          console.error('弹幕数据格式错误:', e);
        }
      };
      socket.onerror = (error) => console.error('WebSocket 错误:', error);
      socket.onclose = () => {
        // 手机切到后台、网络波动都会断开，自动重连，不再弹窗打扰
        console.log('WebSocket 连接关闭，2 秒后重连');
        if (!unmounted) setTimeout(initializeWebSocket, 2000);
      };
    }

    function handleDanmuChange(newDanmu) {
      if (newDanmu.clientId !== clientId.value) {
        const updatedDanmuList = [...danmuList.value, newDanmu];

        if (updatedDanmuList.length > maxDanmuCount.value) {
          danmuList.value = updatedDanmuList.slice(-maxDanmuCount.value);
        } else {
          danmuList.value = updatedDanmuList;
        }
      }
    }

    function checkAvatarAndNickname() {
      if (!nickName.value) {
        showProfileModal.value = true;
      }
    }

    function onCloseModal() {
      if (nickName.value) {
        showProfileModal.value = false;
      } else {
        alert('请先设置昵称');
      }
    }

    function onNickNameInput(e) {
      nickName.value = e.target.value;
    }

    function onTextAreaInput(e) {
      const value = e.target.value;
      const maxLength = maxCharacters.value * 2;

      if (value.replace(/[^\x00-\xff]/g, '**').length > maxLength) {
        msg.value = value.slice(0, maxLength);
        showLimitWarning.value = true;
      } else {
        msg.value = value;
        showLimitWarning.value = false;
      }
    }

    function onTapSubmit() {
      if (!nickName.value) {
        checkAvatarAndNickname();
        return;
      }
      const danmu = {
        nickName: nickName.value,
        text: msg.value,
        clientId: clientId.value,
        timestamp: Date.now()
      };

      if (!socket || socket.readyState !== WebSocket.OPEN) {
        alert('正在连接服务器，请稍后再发送');
        return;
      }
      socket.send(JSON.stringify(danmu));

      // 更新本地弹幕列表
      danmuList.value.push(danmu);
      if (danmuList.value.length > maxDanmuCount.value) {
        danmuList.value.shift();
      }

      msg.value = '';
    }

    function onTextAreaFocus() {
      keyboardHeight.value = 0; // 模拟键盘弹起高度
    }

    function onTextAreaBlur() {
      keyboardHeight.value = 0;
    }

    return {
      msg,
      nickName,
      showProfileModal,
      maxCharacters,
      keyboardHeight,
      danmuList,
      maxDanmuCount,
      maxContainerHeight,
      showLimitWarning,
      onNickNameInput,
      onCloseModal,
      onTextAreaInput,
      onTapSubmit,
      onTextAreaFocus,
      onTextAreaBlur,
    };
  },
};
</script>

<template>
  <header>
  </header>

  <main>
  <div class="background-wrapper">

    <!-- 弹幕容器 -->
    <div class="danmu-container" :style="{ maxHeight: maxContainerHeight + 'px', bottom: keyboardHeight + 'px' }">
      <div v-for="(item, index) in danmuList" :key="index" class="danmu-item danmu-item-animate">
        <!-- 用户头像（可以按需添加） -->
        <!-- <img class="avatar" :src="item.avatarUrl" /> -->
        <span class="nickname">{{ item.nickName }}:</span>
        <span class="message">{{ item.text }}</span>
      </div>
    </div>

    <!-- 登录设置弹框 -->
    <div v-if="showProfileModal" class="login-container">
      <div class="profile-modal">
        <div class="modal-overlay">
          <div class="modal-content">
            <input type="text" class="weui-input" placeholder="请输入昵称" @input="onNickNameInput" style="height: 40px; display: block; box-sizing: border-box;" />
            <button class="modal-submit-button" @click="onCloseModal" style="position: relative; left: 0; top: -20px">确认</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 弹幕输入框 -->
    <div class="danmu-input-bar" :style="{ bottom: keyboardHeight + 'px' }">
      <form @submit.prevent="onTapSubmit" class="input-form">
        <textarea
          class="danmu-textarea"
          placeholder="输入弹幕"
          :rows="1"
          @focus="onTextAreaFocus"
          @blur="onTextAreaBlur"
          @input="onTextAreaInput"
          v-model="msg"
        ></textarea>
        <button class="input-submit-button" type="submit" :disabled="!msg">
          发送
        </button>
      </form>
    </div>

    <!-- 字数限制提示 -->
    <p v-if="showLimitWarning" class="limit-warning">已达到字数限制 (80 字)</p>
  </div>
  </main>
</template>


<style scoped>
/* 设置最外层容器的背景图片 */
.background-wrapper {
  /* 原背景图存放在旧的腾讯云开发存储里，已无法访问，先用渐变色代替。
     有了背景图后放到 public/background.png，再把下面这行换成
     background-image: url('/background.png'); */
  background-image: linear-gradient(160deg, #d9c6e6 0%, #b9cdea 100%);
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 导航栏样式 */
.navigation-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 80px;
  background-color: transparent;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.title {
  color: black;
  font-size: 18px;
  font-weight: bold;
}

/* 模态框遮罩层 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

/* 模态框内容 */
.modal-content {
  background-color: #fff;
  width: 90%;
  max-width: 320px;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
  text-align: center;
  animation: scaleUp 0.3s ease;
}

@keyframes scaleUp {
  from {
    transform: scale(0.8);
  }
  to {
    transform: scale(1);
  }
}

.weui-input {
  width: 100%;
  padding: 10px;
  margin-bottom: 10px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-sizing: border-box;
}

/* 弹幕输入框容器 */
.danmu-input-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  padding: 10px;
  background-color: #f7f7f7;
  border-top: 1px solid #ddd;
  display: flex;
  align-items: center;
  box-shadow: 0 -2px 4px rgba(0, 0, 0, 0.1);
  z-index: 999;
  min-height: 50px;
  padding-bottom: 20px;
}


/* 输入表单样式 */
.input-form {
  position: relative;
  width: 100%;
}

.danmu-textarea {
  width: calc(100% - 80px);
  padding: 8px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  resize: none;
  box-sizing: border-box;
  transform: translateY(10%);
}

/* 模态框确认按钮 */
.modal-submit-button {
  padding: 10px;
  background-color: #007aff;
  color: white;
  font-size: 16px;
  font-weight: bold;
  border-radius: 5px;
  width: 100%;
  margin-top: 40px;
  text-align: center;
}

/* 发送按钮样式 */
.input-submit-button {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  padding: 8px 16px;
  background-color: #007aff;
  color: white;
  border-radius: 4px;
}

/* 弹幕容器样式 */
.danmu-container {
  margin: 80px 0px 60px;
  position: relative;
  overflow-y: auto;
  width: 100%;
  height: calc(100vh - 140px);
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: flex-end;
  padding-bottom: 20px;
  padding-left: 12px;
}

/* 弹幕项样式 */
.danmu-item {
  display: block;
  align-items: center;
  width: auto;
  margin-bottom: 10px;
  opacity: 0;
  animation: fadeInUp 0.5s ease forwards;
  padding: 5px 15px;
  border-radius: 20px;
  background-color: rgba(240, 240, 240, 0.8);
  white-space: normal;
  word-break: break-word;
  max-width: 96%;
  box-sizing: border-box;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}


.nickname {
  font-weight: bold;
  color: #333;
  margin-right: 5px;
}

.message {
  color: #333;
  word-break: break-word;
  white-space: normal;
}

</style>
