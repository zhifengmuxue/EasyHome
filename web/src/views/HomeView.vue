<script setup lang="ts">
import { defineComponent } from "vue";
import { ref, onUnmounted, onMounted, computed } from "vue";
import { marked } from "marked";
import { markedHighlight } from "marked-highlight";
import hljs from "highlight.js";
import "highlight.js/styles/base16/darcula.css";
import HouseMiniCard from "@/components/HouseMiniCard.vue";
import HouseDetailModal from "@/components/HouseDetail.vue";
interface Message {
  role: "user" | "assistant";
  content: string;
}

interface StreamResponse {
  content: string;
  role: string;
  finished: boolean;
  aggregationMessage: string;
}

defineComponent({
  name: "HomeView",
});

const userInput = ref("");
const messages = ref<Message[]>([]);
const isLoading = ref(false);
const isTyping = ref(false);
// 会话标识符，在页面加载时生成，之后所有请求都使用此标识符
const sessionId = ref("");

let controller: AbortController | null = null;

const resetAllStates = () => {
  isLoading.value = false;
  isTyping.value = false;
  controller = null;
};

// 在组件挂载时生成会话ID和初始化测试数据
onMounted(() => {
  sessionId.value = crypto.randomUUID();
  console.log("创建新会话:", sessionId.value);
  
  // 添加一些测试房源ID，实际使用时可以移除
  // 在开发环境中添加测试数据
});

const sendMessage = async (): Promise<void> => {
  if (!userInput.value.trim() || isLoading.value) return;

  // 添加用户消息
  const newUserMessage: Message = {
    role: "user",
    content: userInput.value,
  };

  messages.value.push(newUserMessage);

  // 准备发送请求
  isLoading.value = true;
  isTyping.value = true;

  // 创建新的AbortController用于取消请求
  controller = new AbortController();

  try {
    // 直接使用用户输入作为消息内容，历史记录由后端维护
    let prompt = userInput.value;

    // 创建请求对象
    const requestBody = {
      message: prompt,
      uuid: sessionId.value, // 传递会话ID，后端据此维护历史记录
    };

    // 调用API的流式接口
    const response = await fetch("/api/chat/stream", {
      method: "POST",
      signal: controller.signal,
      headers: {
        "Content-Type": "application/json",
        Accept: "text/event-stream",
      },
      body: JSON.stringify(requestBody),
    });

    // 检查响应状态
    if (!response.ok) {
      throw new Error("网络请求失败");
    }

    // 添加空的助手回复
    messages.value.push({ role: "assistant", content: "" });

    // 清空用户输入
    userInput.value = "";

    // 使用原生的流式API处理SSE格式
    const reader = response.body?.getReader();
    if (!reader) throw new Error("无法读取响应流");

    // 读取流数据
    const decoder = new TextDecoder();

    try {
      while (true) {
        const { done, value } = await reader.read();

        if (done) {
          resetAllStates();
          break;
        }

        if (value) {
          // 解码接收到的数据
          const chunk = decoder.decode(value, { stream: true });
          // 处理SSE格式的数据，移除"data:"前缀并解析JSON
          const lines = chunk.split("\n");
          for (const line of lines) {
            let jsonContent = line.trim();
            if (jsonContent.startsWith("data:"))
              jsonContent = jsonContent.substring(5).trim();

            if (jsonContent) {
              try {
                const parsedData: StreamResponse = JSON.parse(jsonContent);

                // 更新最新的AI消息内容为聚合消息
                // 指针指向messages的最后一个元素
                const lastMessage = messages.value[messages.value.length - 1];
                lastMessage.content = parsedData.aggregationMessage;
                
                // 解析消息中的房屋ID
                renderhousesCards(parsedData.aggregationMessage);

                // 如果标记为完成，重置状态
                if (parsedData.finished) {
                  resetAllStates();
                  break;
                }
              } catch (e) {
                console.error("JSON解析错误:", e);
              }
            }
          }
        }
      }

      // 处理流结束后可能的最后一个块
      const finalChunk = decoder.decode();
      if (finalChunk) {
        const lines = finalChunk.split("\n");
        for (const line of lines) {
          if (line.startsWith("data:")) {
            const jsonContent = line.substring(5).trim();
            if (jsonContent) {
              try {
                const parsedData: StreamResponse = JSON.parse(jsonContent);
                const lastMessage = messages.value[messages.value.length - 1];
                lastMessage.content = parsedData.aggregationMessage;
                
                // 解析消息中的房屋ID
                renderhousesCards(parsedData.aggregationMessage);
              } catch (e) {
                // 忽略解析错误
              }
            }
          }
        }
      }
    } finally {
      resetAllStates();
    }
  } catch (error: any) {
    if (error.name !== "AbortError") {
      console.error("流式请求出错:", error);
      messages.value.push({
        role: "assistant",
        content: "抱歉，发生了错误，请重试。",
      });
    }
    resetAllStates();
  }
};

const cancelRequest = () => {
  if (controller) {
    controller.abort();
    resetAllStates();
  }
};

// 创建新会话
const createNewSession = () => {
  messages.value = [];
  sessionId.value = crypto.randomUUID();
  console.log("创建新会话:", sessionId.value);
};

// 表单提交处理
const handleSubmit = (e: Event) => {
  e.preventDefault(); // 阻止表单默认提交行为
  sendMessage();
};

// 组件卸载时取消可能存在的未完成请求
onUnmounted(() => {
  cancelRequest();
});

marked.use(
  markedHighlight({
    langPrefix: "hljs language-",
    highlight(code, lang) {
      const language = hljs.getLanguage(lang) ? lang : "shell";
      return hljs.highlight(code, { language }).value;
    },
  })
);

const renderedContent = (message: Message) => {
  return marked.parse(message.content);
};

const housesRecommend = ref<number[]>([]);

// 模态框状态
const showDetailModal = ref(false);
const selectedHouse = ref<any>(null);

// 查看房源详情
const viewHouseDetail = (house: any) => {
  selectedHouse.value = house;
  showDetailModal.value = true;
};

// 关闭详情模态框
const closeDetailModal = () => {
  showDetailModal.value = false;
};

// 切换收藏状态
const toggleFavorite = async (houseId: number) => {
  console.log('切换收藏状态:', houseId);
  // 这里可以添加收藏/取消收藏的逻辑
  // 可以复用SearchView中的收藏逻辑
};

// 控制推荐房源区域的显示与隐藏
const showRecommendations = computed(() => housesRecommend.value.length > 0);

// 限制显示的推荐房源数量，最多显示4个
const displayedHouseIds = computed(() => {
  return housesRecommend.value.slice(0, 4);
});

function renderhousesCards(content: string) {
  // 解析content，获取housesRecommend
  // 所有housesRecommend的值都为数字，以类似于 [^1] [^2] [^3] 的形式给出
  // 将housesRecommend的值都转换为数字，并存储到housesRecommend中
  // 和原有的值去重后，将新值插入到housesRecommend的头部
  const regex = /\[\^(\d+)\]/g;
  let match;
  const newHouseIds: number[] = [];
  
  // 提取所有匹配的数字
  while ((match = regex.exec(content)) !== null) {
    const houseId = parseInt(match[1], 10);
    if (!isNaN(houseId) && !newHouseIds.includes(houseId)) {
      newHouseIds.push(houseId);
    }
  }
  
  if (newHouseIds.length > 0) {
    // 过滤掉已经存在的ID，避免重复
    const uniqueNewIds = newHouseIds.filter(id => !housesRecommend.value.includes(id));
    
    // 将新的ID添加到数组头部
    housesRecommend.value = [...uniqueNewIds, ...housesRecommend.value];
  }
}
</script>

<template>
  <main class="py-8 bg-gray-50 h-full">
    <div class="max-w-[80%] mx-auto px-4 sm:px-6 lg:px-8">
      <!-- 消息区域 -->
      <div
        v-show="messages.length > 0"
        class="bg-white shadow-sm rounded-xl mb-6 p-8 h-full overflow-y-auto"
      >
        <div
          v-if="messages.length === 0"
          class="flex items-center justify-center h-full"
        >
          <div class="text-center text-gray-400">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-16 w-16 mx-auto mb-4 text-gray-300"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M8 10h.01M12 10h.01M16 10h.01M9 16H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-5l-5 5v-5z"
              />
            </svg>
            <p class="text-xl">开始对话吧...</p>
          </div>
        </div>
        <div class="space-y-6">
          <div
            v-for="(message, index) in messages"
            :key="index"
            class="flex"
            :class="message.role === 'user' ? 'justify-end' : 'justify-start'"
          >
            <div
              class="rounded-2xl px-6 py-4 shadow-sm"
              :class="
                message.role === 'user'
                  ? 'bg-blue-100 text-white'
                  : 'bg-gray-100 text-gray-800'
              "
            >
              <div class="prose" v-html="renderedContent(message)"></div>
            </div>
          </div>
          <div v-if="isTyping" class="flex justify-start">
            <div
              class="bg-gray-100 rounded-2xl px-6 py-4 shadow-sm text-gray-500"
            >
              <span class="dots-loading">AI正在思考</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 推荐房源区域 -->
      <div v-if="showRecommendations" class="bg-white shadow-sm rounded-xl mb-6 p-6">
        <div class="flex justify-between items-center mb-4">
          <h2 class="text-lg font-medium text-gray-900">为您推荐的房源</h2>
          <router-link 
            to="/search" 
            class="text-sm text-blue-600 hover:text-blue-800 flex items-center"
          >
            查看更多
            <svg class="h-4 w-4 ml-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
            </svg>
          </router-link>
        </div>
        
        <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-4">
          <div v-for="houseId in displayedHouseIds" :key="houseId">
            <HouseMiniCard 
              :houseId="houseId" 
              @viewDetail="viewHouseDetail" 
              @toggleFavorite="toggleFavorite"
            />
          </div>
        </div>
      </div>

      <!-- 输入区域 -->
      <form @submit="handleSubmit" class="bg-white shadow-sm rounded-xl p-6">
        <div class="flex items-center">
          <input
            v-model="userInput"
            class="flex-1 border border-gray-200 rounded-lg px-5 py-4 mr-3 text-base focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            placeholder="输入您的问题..."
            :disabled="isLoading"
          />
          <button
            v-if="!isLoading"
            type="button"
            @click="createNewSession"
            class="mr-3 bg-gray-200 hover:bg-gray-300 text-gray-700 rounded-lg px-4 py-4 transition-colors duration-200"
          >
            新对话
          </button>
          <button
            type="submit"
            :disabled="isLoading || !userInput.trim()"
            class="bg-blue-600 hover:bg-blue-700 text-white rounded-lg px-8 py-4 font-medium text-base transition-colors duration-200 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            发送
          </button>
          <button
            v-if="isLoading"
            type="button"
            @click="cancelRequest"
            class="ml-3 bg-gray-500 hover:bg-gray-600 text-white rounded-lg px-6 py-4 transition-colors duration-200"
          >
            取消
          </button>
        </div>
      </form>
    </div>
    
    <!-- 房源详情模态框 -->
    <HouseDetailModal 
      :show="showDetailModal" 
      :house="selectedHouse" 
      @close="closeDetailModal"
      @favoriteChange="(houseId, isFav) => toggleFavorite(houseId)"
    />
  </main>
</template>

<style scoped>
.dots-loading:after {
  content: " .";
  animation: dots 1.5s steps(5, end) infinite;
}

@keyframes dots {
  0%,
  20% {
    content: " .";
  }
  40% {
    content: " ..";
  }
  60% {
    content: " ...";
  }
  80%,
  100% {
    content: "";
  }
}
</style>
