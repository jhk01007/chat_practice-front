<template>
    <v-container>
        <v-row justify="center">
            <v-col cols="12" md="8">
                <v-card>
                    <v-card-titile class="text-center text-h5">
                        채팅
                    </v-card-titile>
                    <v-card-text>
                        <div class="chat-box">
                            <div 
                            v-for="(msg, index) in messages"
                            :key="index"
                            :class="['chat-message', msg.senderEmail === this.senderEmail ? 'sent' : 'received']"
                            >
                                <strong>{{ msg.senderEmail }}: </strong> {{ msg.message }}
                            </div>
                        </div>
                        <v-text-field
                            v-model="newMessage"
                            label="메시지 입력"
                            @keyup.enter="sendMessage"
                        />
                        <v-btn color="primary" block @click="sendMessage">전송</v-btn>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
import SockJS from 'sockjs-client';
import Stomp from 'webstomp-client';
import axios from 'axios';


    export default {
        data() {
            return {
                // ws: null,
                messages: [],
                newMessage: "",
                stompClient: null,
                token: "",
                senderEmail: null,
                roomId: null,
            }
        },
        async created() {
            this.senderEmail = localStorage.getItem("email");
            this.roomId = this.$route.params.roomId;
            // 채팅방에 이전 메시지 조회
            const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/chat/history/${this.roomId}`)
            this.messages = response.data;
            // 소켓 연결
            this.connectWebSocket();
        },
        // 사용자가 현재 라우트에서 다른 라우트로 이동하려고 할 때 호출되는 함수
        beforeRouteLeave (to, from, next) {
            this.disconnectWebSocket()
            next();
        },
        // 화면을 완전히 꺼버렸을 때
        beforeUnmount() {
            this.disconnectWebSocket()
        },
        methods: {
            connectWebSocket() {

                // 이미 연결이 되어 있으면 새로 연결객체를 만들지 않음
                if(this.stompClient && this.stompClient.connected) return;

                // sockJs는 websocket을 내장한 향상된 js 라이브러리. http 엔드포인트 사용.
                const sockJs = new SockJS(`${process.env.VUE_APP_API_BASE_URL}/connect`)
                this.stompClient = Stomp.over(sockJs);
                this.token = localStorage.getItem("token");

                this.stompClient.connect({
                    Authorization: `Bearer ${this.token}`
                },
                    () => {
                        this.stompClient.subscribe(`/topic/${this.roomId}`, (message) => {
                            console.log(message)
                            const parseMessage = JSON.parse(message.body);
                            this.messages.push(parseMessage);
                            this.scrollToBottom();
                        }, {Authorization: `Bearer ${this.token}`})
                    }
                )
            },

            sendMessage() {
                if(this.newMessage.trim() == "") return;
                const message = {
                    senderEmail: this.senderEmail,
                    message: this.newMessage
                }
                this.stompClient.send(`/publish/${this.roomId}`, JSON.stringify(message));
                this.newMessage = ""
            },

            scrollToBottom() {
                this.$nextTick(() => {
                    const chatBox = this.$el.querySelector(".chat-box");
                    chatBox.scrollTop = chatBox.scrollHeight;
                })
            },

            async disconnectWebSocket() {
                await axios.post(`${process.env.VUE_APP_API_BASE_URL}/chat/room/${this.roomId}/read`)
                if(this.stompClient && this.stompClient.connected) {
                    this.stompClient.unsubscribe(`/topic/${this.roomId}`);
                    this.stompClient.disconnect();
                }
            }
        }
    }
</script>
<style>
.chat-box{
    height: 300px;
    overflow-y: auto;
    border: 1px solid #ddd;
    margin-bottom: 10px;
}

.chat-message {
    margin-bottom: 10px;
}

.sent {
    text-align: right;
}

.received {
    text-align: left;
}
</style>