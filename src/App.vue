<template>
  <div class="app-container">
    <div class="chat-header">
      <h1>文案生成</h1>
    </div>

    <div class="chat-container">
      <div class="message-list" ref="messageList">
        <div
          v-for="(message, index) in messages"
          :key="index"
          :class="[
            'message-item',
            message.isUser ? 'user-message' : 'bot-message',
          ]"
        >
          <div class="message-content">
            {{ message.content }}
            <span
              v-if="
                isLoading && index === messages.length - 1 && !message.isUser
              "
              class="loading-indicator"
            >
              <span class="dot"></span>
              <span class="dot"></span>
              <span class="dot"></span>
            </span>
          </div>
          <div class="message-time">{{ formatTime(message.timestamp) }}</div>
        </div>
      </div>
    </div>

    <div class="input-container">
      <input
        v-model="inputMessage"
        @keyup.enter="sendMessage"
        placeholder="请输入消息..."
        class="message-input"
        :disabled="isLoading"
      />
      <button
        @click="sendMessage"
        class="send-button"
        :disabled="!inputMessage.trim() || isLoading"
      >
        发送
      </button>
    </div>
  </div>
</template>

<script>
import { CozeAPI } from "@coze/api";

export default {
  name: "App",
  data() {
    return {
      inputMessage: "",
      messages: [],
      isLoading: false,
      // Coze API 客户端实例
      apiClient: null,
      // 当前正在接收的机器人消息ID
      currentBotMessageId: null,
    };
  },
  mounted() {
    this.loadMessages();
    // 初始化Coze API客户端
    this.initApiClient();
  },
  methods: {
    // 初始化Coze API客户端
    initApiClient() {
      this.apiClient = new CozeAPI({
        token:
          "cztei_hwzguYFV4jbonUfHakmlHPXdyEz5VsLTQ80i9b4Va5RYnNX0wChBGe85I2tB10buw",
        baseURL: "https://api.coze.cn",
      });
    },

    // 加载本地存储的消息
    loadMessages() {
      const savedMessages = localStorage.getItem("chatMessages");
      if (savedMessages) {
        this.messages = JSON.parse(savedMessages);
      }
    },

    // 保存消息到本地存储
    saveMessages() {
      localStorage.setItem("chatMessages", JSON.stringify(this.messages));
    },

    // 发送消息
    async sendMessage() {
      if (!this.inputMessage.trim() || this.isLoading) return;

      const message = this.inputMessage.trim();
      this.inputMessage = "";

      // 添加用户消息
      this.addMessage(message, true);

      // 滚动到底部
      this.scrollToBottom();

      try {
        this.isLoading = true;

        // 创建一个空的机器人消息，用于实时更新流式响应
        const botMessage = this.addMessage("", false);
        this.currentBotMessageId = this.messages.length - 1;

        // 调用Coze API的流式接口
        await this.callCozeApiWithStream(message);
      } catch (error) {
        console.error("API调用错误:", error);

        // 如果有当前机器人消息，更新为错误信息
        if (this.currentBotMessageId !== null) {
          this.messages[this.currentBotMessageId].content =
            "很抱歉，智能服务暂时不可用，请稍后再试。";
          this.currentBotMessageId = null;
          this.saveMessages();
        } else {
          // 否则添加新的错误消息
          this.addMessage("很抱歉，智能服务暂时不可用，请稍后再试。", false);
        }
      } finally {
        this.isLoading = false;
        this.scrollToBottom();
      }
    },

    // 调用Coze API的流式接口
    async callCozeApiWithStream(userMessage) {
      try {
        console.log("开始调用Coze API流式接口");

        // 构建消息参数
        const additionalMessages = [
          {
            content: userMessage,
            content_type: "text",
            role: "user",
            type: "question",
          },
        ];

        console.log("API请求参数:", {
          bot_id: "7576901197736067099",
          user_id: "123456789",
          additional_messages: additionalMessages,
        });

        // 调用流式API
        const res = await this.apiClient.chat.stream({
          bot_id: "7576901197736067099",
          user_id: "123456789",
          additional_messages: additionalMessages,
        });

        console.log("成功获取流式响应");

        // 处理流式响应
        for await (const chunk of res) {
          console.log("接收到流式响应chunk:", chunk);

          // 检查是否有currentBotMessageId，确保机器人消息已创建
          if (this.currentBotMessageId === null) {
            this.addMessage("", false);
            this.currentBotMessageId = this.messages.length - 1;
            console.log("创建新的机器人消息，ID:", this.currentBotMessageId);
          }

          // 更新机器人消息内容
          if (chunk.data && chunk.data.content) {
            console.log("更新机器人消息内容:", chunk.data.content);
            // 拼接内容而不是替换，实现流式效果
            this.messages[this.currentBotMessageId].content +=
              chunk.data.content;
            this.scrollToBottom();
          } else if (chunk.message && chunk.message.content) {
            console.log("更新机器人消息内容:", chunk.message.content);
            this.messages[this.currentBotMessageId].content +=
              chunk.message.content;
            this.scrollToBottom();
          } else if (chunk.answer && chunk.answer.content) {
            console.log("使用answer字段更新内容:", chunk.answer.content);
            this.messages[this.currentBotMessageId].content +=
              chunk.answer.content;
            this.scrollToBottom();
          } else {
            console.log(
              "chunk中没有找到消息内容，chunk结构:",
              Object.keys(chunk)
            );
          }
        }

        console.log("流式响应处理完成");

        // 流式响应结束，重置当前消息ID
        this.currentBotMessageId = null;
        this.saveMessages();
      } catch (error) {
        console.error("API调用错误详情:", error);
        throw error;
      }
    },

    // 添加消息到列表
    addMessage(content, isUser) {
      const message = {
        content,
        isUser,
        timestamp: new Date(),
      };

      this.messages.push(message);

      // 保持最多10条消息
      if (this.messages.length > 10) {
        this.messages.shift();
      }

      // 保存到本地存储
      this.saveMessages();

      return message;
    },

    // 滚动到底部
    scrollToBottom() {
      this.$nextTick(() => {
        if (this.$refs.messageList) {
          this.$refs.messageList.scrollTop =
            this.$refs.messageList.scrollHeight;
        }
      });
    },

    // 格式化时间
    formatTime(timestamp) {
      const date = new Date(timestamp);
      const hours = date.getHours().toString().padStart(2, "0");
      const minutes = date.getMinutes().toString().padStart(2, "0");
      return `${hours}:${minutes}`;
    },
  },
};
</script>

<style scoped>
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  max-width: 414px;
  margin: 0 auto;
  background-color: #fff;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.chat-header {
  background-color: #e9e9e9;
  padding: 16px;
  text-align: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.chat-header h1 {
  margin: 0;
  font-size: 18px;
  font-weight: 500;
}

.chat-container {
  flex: 1;
  overflow: hidden;
}

.message-list {
  height: 100%;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.message-item {
  max-width: 70%;
  word-wrap: break-word;
  animation: slideIn 0.3s ease;
}

.user-message {
  align-self: flex-end;
}

.bot-message {
  align-self: flex-start;
}

.message-content {
  padding: 12px 16px;
  border-radius: 18px;
  font-size: 15px;
  line-height: 1.4;
  position: relative;
}

.user-message .message-content {
  background-color: #07c160;
  color: white;
  border-bottom-right-radius: 4px;
}

.bot-message .message-content {
  background-color: #f2f2f2;
  color: #333;
  border-bottom-left-radius: 4px;
}

.message-time {
  font-size: 11px;
  color: #999;
  margin-top: 4px;
  text-align: right;
}

.bot-message .message-time {
  text-align: left;
}

/* 加载指示器样式 */
.loading-indicator {
  display: inline-flex;
  align-items: center;
  margin-left: 5px;
}

.dot {
  width: 6px;
  height: 6px;
  background-color: #999;
  border-radius: 50%;
  margin: 0 2px;
  animation: loadingDots 1.4s infinite ease-in-out both;
}

.dot:nth-child(1) {
  animation-delay: -0.32s;
}

.dot:nth-child(2) {
  animation-delay: -0.16s;
}

@keyframes loadingDots {
  0%,
  80%,
  100% {
    transform: scale(0);
  }
  40% {
    transform: scale(1);
  }
}

.input-container {
  display: flex;
  padding: 12px 16px;
  background-color: #fff;
  border-top: 1px solid #eee;
  gap: 12px;
}

.message-input {
  flex: 1;
  padding: 12px 16px;
  border: 1px solid #ddd;
  border-radius: 20px;
  font-size: 15px;
  outline: none;
  transition: border-color 0.3s;
}

.message-input:focus:not(:disabled) {
  border-color: #07c160;
}

.message-input:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

.send-button {
  padding: 12px 24px;
  background-color: #07c160;
  color: white;
  border: none;
  border-radius: 20px;
  font-size: 15px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.send-button:hover:not(:disabled) {
  background-color: #06ad56;
}

.send-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 滚动条样式 */
.message-list::-webkit-scrollbar {
  width: 4px;
}

.message-list::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.message-list::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 2px;
}

.message-list::-webkit-scrollbar-thumb:hover {
  background: #555;
}
</style>
