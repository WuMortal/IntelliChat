<template>
  <div class="chat-container">
    <div class="chat-messages" ref="messageContainer">
      <div
        v-for="(message, index) in messages"
        :key="index"
        class="message"
        :class="message.role"
      >
        <div class="avatar" v-if="message.role === 'assistant'">
          <img src="https://api.dicebear.com/7.x/bottts/svg?seed=assistant" alt="Assistant" />
        </div>
        <div class="message-content">
          <div v-if="message.role === 'assistant'" class="typing-content">
            <span
              v-if="
                isThinking && index === messages.length - 1 && !message.content
              "
              class="thinking"
              >思考中...</span
            >
            <template v-else>
              <div v-if="index === messages.length - 1">
                <div v-if="currentReasoningContent" class="reasoning-content">
                  {{ currentReasoningContent }}
                </div>
                <div v-html="md.render(currentTypingMessage)"></div>
              </div>
              <div v-else>
                <div v-if="message.reasoning_content" class="reasoning-content">
                  {{ message.reasoning_content }}
                </div>
                <div v-html="md.render(message.content)"></div>
              </div>
            </template>
          </div>
          <div v-else>
            {{ message.content }}
          </div>
        </div>
        <div class="avatar" v-if="message.role === 'user'">
          <img src="https://api.dicebear.com/7.x/avataaars/svg?seed=user" alt="User" />
        </div>
      </div>
    </div>
    <div class="input-container">
      <textarea
        v-model="userInput"
        @keydown.enter.prevent="sendMessage"
        placeholder="请输入内容..."
        rows="3"
      ></textarea>
      <button @click="sendMessage" :disabled="isLoading">发送</button>
    </div>
  </div>
</template>

<script>
import MarkdownIt from "markdown-it";

export default {
  name: "ChatView",
  data() {
    return {
      md: new MarkdownIt(),
      messages: [],
      userInput: "",
      isLoading: false,
      isThinking: false,
      apiKey: process.env.VUE_APP_DEEPSEEK_API_KEY || "",
      apiEndpoint: "http://10.3.106.85:11434/api/generate",
      typingSpeed: 50, // 打字速度（毫秒）
      currentTypingMessage: null,
      currentReasoningContent: null,
      isStartReasoningContent: false,
    };
  },
  methods: {
    async sendMessage() {
      if (!this.userInput.trim() || this.isLoading) return;

      // 添加用户消息
      this.messages.push({
        role: "user",
        content: this.userInput.trim(),
      });

      this.userInput = "";
      this.isLoading = true;
      this.isThinking = true;

      // 添加助手消息占位
      const assistantMessage = {
        role: "assistant",
        content: "",
      };
      this.messages.push(assistantMessage);
      await this.$nextTick();
      this.scrollToBottom();
      try {
        this.currentTypingMessage = "";
        this.currentReasoningContent = "";
        this.isStartReasoningContent = false;
        await this.callDeepseekAPI(assistantMessage);
      } catch (error) {
        console.error("Error:", error);
        assistantMessage.content = "抱歉，发生了错误，请稍后重试。";
      } finally {
        this.isLoading = false;
        this.$nextTick(() => {
          this.scrollToBottom();
        });
      }
    },

    async callDeepseekAPI(assistantMessage) {
      const response = await fetch(this.apiEndpoint, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          model: "deepseek-r1:1.5b",
          prompt: this.messages.map((msg) => msg.content).join("\n"),
          stream: true,
          options: {
            temperature: 0.7,
            top_p: 0.9,
          },
        }),
      });

      if (!response.ok) {
        throw new Error(`API请求失败: ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      try {
        this.isThinking = false;
        let reading = true;
        while (reading) {
          const { done, value } = await reader.read();
          if (done) {
            reading = false;
            break;
          }

          const chunk = decoder.decode(value);
          const lines = chunk.split("\n").filter((line) => line.trim() !== "");
          for (const line of lines) {
            try {
              const parsed = JSON.parse(line);
              let content = parsed.response || "";

              let reasoningContent = "";

              if (content.match("</think>")) {
                reasoningContent = content
                  .replace("<think>", "")
                  .replace("</think>", "")
                  .trim();
                this.isStartReasoningContent = false;
                content = "";
              }

              if (content.match("<think>") || this.isStartReasoningContent) {
                reasoningContent = content
                  .replace("<think>", "")
                  .replace("</think>", "").trim();
                this.isStartReasoningContent = true;
                content = "";
              }

              if (content || reasoningContent) {
                if (content) {
                  assistantMessage.content += content;
                  this.currentTypingMessage += content;
                }
                if (reasoningContent) {
                  if (!assistantMessage.reasoning_content) {
                    assistantMessage.reasoning_content = "";
                  }
                  assistantMessage.reasoning_content += reasoningContent;
                  this.currentReasoningContent += reasoningContent;
                }
                await this.$nextTick();
                this.scrollToBottom();
              }
            } catch (e) {
              console.error("解析响应数据失败:", e);
            }
          }
        }
      } finally {
        reader.releaseLock();
      }
    },

    scrollToBottom() {
      if (this.$refs.messageContainer) {
        this.$refs.messageContainer.scrollTop =
          this.$refs.messageContainer.scrollHeight;
      }
    },
  },
};
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  box-sizing: border-box;
  position: fixed;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
  margin-bottom: 20px;
  box-sizing: border-box;
}

.message {
  margin-bottom: 20px;
  display: flex;
  flex-direction: row;
  align-items: flex-start;
}

.message.user {
  /* flex-direction: row-reverse; */
  justify-content: flex-end;
}

.avatar {
  width: 40px;
  height: 40px;
  margin: 0 10px;
  border-radius: 50%;
  overflow: hidden;
  flex-shrink: 0;
}

.avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.message-content {
  max-width: 70%;
  padding: 12px 16px;
  border-radius: 8px;
  background: white;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.reasoning-content {
  margin-top: 10px;
  padding: 8px 12px;
  background: #f8f9fa;
  border-left: 4px solid #007aff;
  font-style: italic;
  color: #495057;
  border-radius: 4px;
  font-size: 0.9em;
  margin-bottom: 10px;
}

.user .message-content {
  background: #007aff;
  color: white;
}

.input-container {
  display: flex;
  gap: 10px;
}

textarea {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  resize: none;
  font-family: inherit;
}

button {
  padding: 0 20px;
  background: #007aff;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background: #0056b3;
}

.thinking {
  display: inline-block;
  animation: thinking 1.4s infinite;
  position: relative;
}

.thinking::after {
  content: "";
  animation: dots 1.4s linear infinite;
}

@keyframes thinking {
  0%,
  20% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}

@keyframes dots {
  0% {
    content: "";
  }
  25% {
    content: ".";
  }
  50% {
    content: "..";
  }
  75% {
    content: "...";
  }
  100% {
    content: "";
  }
}
</style>