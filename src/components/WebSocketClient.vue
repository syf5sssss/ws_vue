<script setup>
import { ref, nextTick, defineProps } from 'vue';
import Button from 'primevue/button';
import Textarea from 'primevue/textarea';
import DataView from 'primevue/dataview';
import Tag from 'primevue/tag';
import ScrollPanel from 'primevue/scrollpanel';
import InputText from 'primevue/inputtext';

const props = defineProps({
    serverIp: {
        type: String,
        default: 'localhost'
    },
    serverPort: {
        type: Number,
        default: 9999
    },
    maxMessages: {
        type: Number,
        default: 1000
    }
});

const msg = ref("");
const ip = ref(props.serverIp);
const port = ref(props.serverPort);
const chatting_ls = ref([]);
const server_ls = ref([]);
const scrollChatRef = ref(null);
const scrollServerRef = ref(null);
let socket = null;

// 滚动到底部方法
const scrollToBottomChatting = () => {
    nextTick(() => {
        const container = scrollChatRef.value?.$el.querySelector('.p-scrollpanel-content')
        if (container) {
            container.scrollTo({
                top: container.scrollHeight,
                behavior: 'smooth'
            })
        }
    })
}
const scrollToBottomServer = () => {
    nextTick(() => {
        const container = scrollServerRef.value?.$el.querySelector('.p-scrollpanel-content')
        if (container) {
            // container.scrollTop = container.scrollHeight
            // 或使用平滑滚动（需浏览器支持）
            container.scrollTo({
                top: container.scrollHeight,
                behavior: 'smooth'
            })
        }
    })
}

function open() {
    const wsUrl = `ws://${ip.value}:${port.value}`;
    socket = new WebSocket(wsUrl);

    socket.addEventListener('open', (event) => {
        logServerMessage('WebSocket连接已打开');
    });

    socket.addEventListener('message', (event) => {
        chatting_ls.value.push(event);
        scrollToBottomChatting();
    });

    socket.addEventListener('close', (event) => {
        logServerMessage('WebSocket连接已关闭');
    });

    socket.addEventListener('error', (error) => {
        logServerMessage(`WebSocket错误: ${error.message}`);
    });
}

function logServerMessage(message) {
    server_ls.value.push({ value: message });
    if (server_ls.value.length > props.maxMessages) {
        server_ls.value.shift();
    }
    scrollToBottomServer();
}
function stop() {
    let m = "关闭服务 端口:" + port.value;
    socket.close();
    server_ls.value.push({ value: m });
    scrollToBottomServer();
}

function send() {
    socket.send(msg.value);
    chatting_ls.value.push({ tag: "send", data: msg.value });
    scrollToBottomChatting();
}

const formatData = (data) => {
    try {
        console.log(data);
        // 尝试解析为 JSON
        const parsedData = JSON.parse(data);
        // 如果解析成功，将其转换为格式化后的 JSON 字符串
        return JSON.stringify(parsedData, null, 2);
    } catch (error) {
        // 如果解析失败，说明不是有效的 JSON 字符串，直接返回原数据
        return data;
    }
};

const showMessageContent = (content) => {
    msg.value = formatData(content);
};

const getItemClass = (tag) => {
    return tag === 'send' ? 'text-right' : 'text-left';
};
// 暴露公共方法
defineExpose({
    connect: open,
    disconnect: stop,
    sendMessage: send
});
</script>

<template>
    <div class="flex items-center p-1">
        <InputText style="width: 70%;" type="text" v-model="ip" class="mt-0 ml-0"></InputText>
        <InputText style="width: 30%;" type="text" v-model="port" class="mt-0 ml-0"></InputText>
    </div>

    <div class="flex items-center p-1">
        <Button @click="open" style="width: 50%;" class="ml-2 ">连接</Button>
        <Button @click="stop" style="width: 50%;" class="ml-2 ">关闭</Button><br />
    </div>
    <div style="width: 99%;">
        <Textarea v-model="msg" rows="1" fluid />
        <Button @click="send" fluid>发送</Button>
    </div>
    <div>
        <DataView :value="chatting_ls">
            <template #empty>
                <div></div>
            </template>
            <template #list="slotProps">
                <ScrollPanel style="width: 100%; height: 50vh" ref="scrollChatRef">
                    <div v-for="(item, index) in slotProps.items" :key="index" :class="getItemClass(item.tag)">
                        <Tag :severity="item.tag === 'send' ? 'primary' : 'info'" :value="item.data"
                            @dblclick="showMessageContent(item.data)" class="mt-1" />
                    </div>
                </ScrollPanel>
            </template>
        </DataView>
    </div>
    <div>
        <DataView :value="server_ls">
            <template #empty>
                <div></div>
            </template>
            <template #list="slotProps">
                <ScrollPanel style="width: 100%; height: 15vh" ref="scrollServerRef">
                    <div v-for="(item, index) in slotProps.items" :key="index">
                        <Tag severity="secondary" class="mt-1" @dblclick="showMessageContent(item.value)">{{
                            item.value }}
                        </Tag>
                    </div>
                </ScrollPanel>
            </template>
        </DataView>
    </div>
</template>

<style scoped>
.grid-datatable {
    width: auto;
}

.text-right {
    text-align: right;
}

.text-left {
    text-align: left;
}
</style>