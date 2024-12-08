<script setup>
import {ref, onMounted, onBeforeUnmount, nextTick, watch} from 'vue';
import axios from 'axios';
import { Stomp } from '@stomp/stompjs';
import { v4 as uuidv4 } from 'uuid';
import dayjs from 'dayjs';
import {useAuthStore} from "@/stores/auth.js";

const props = defineProps({
  chat: Object
});
const messages = ref([]);
const newMessage = ref('');
const messagesContainer = ref(null);
const stompClient = ref(null);
const sendUsername = ref(props.chat.sendUserNickname);
const receiveUsername = ref(props.chat.receiveUserNickname);
const isConnected = ref(false);
const emit = defineEmits(['goBack']);
const currentUserSeq = useAuthStore().userSeq;
const userNickname = ref(null);

const readUser = async () => {
  try {
    const response = await axios.get(`http://localhost:8089/api/v1/user/${currentUserSeq}`, {
      headers: {
        Authorization: `Bearer ${localStorage.getItem('accessToken')}`,
      },
    });
    userNickname.value = response.data.userNickname;
  } catch (error) {
    console.error("유저 정보 가져오기 실패:", error);
  }
};

onMounted(async () => {
  await loadChatHistory(props.chat.chatSeq);
  await readUser();

  connectWebSocket();
});

async function loadChatHistory(chatId) {
  console.log(chatId);
  try {
    const response = await fetch(`http://localhost:8089/api/v1/user/chat/message/${chatId}`, {
      headers: {
        Authorization: `Bearer ${localStorage.getItem('accessToken')}`,
      },
    });
    const data = await response.json();
    messages.value = data.map((message) => ({
      chatMessageSeq: message.chatMessageSeq,
      chatRoomSeq: message.chatRoomSeq,
      senderNickName: message.senderNickName,
      sendSeq: message.sendSeq,
      receiveSeq: message.receiveSeq,
      text: message.message,
      type: message.type == "ENTER" ? "ENTER" : currentUserSeq  == message.sendSeq ? "SENT" : "RECEIVED",
      regDate: message.regDate,
      readYn: message.readYn
    }));
    scrollToBottom();
  } catch (error) {
    console.error("Failed to load chat history:", error);
  }
}

onBeforeUnmount(() => {
  disconnect();
});

const suggestMessage = ref("");

function connectWebSocket() {
  const websocket = new WebSocket("ws://localhost:8089/stomp/chat");
  stompClient.value = Stomp.over(websocket);

  stompClient.value.connect({}, (frame) => {
    isConnected.value = true;

    // 메시지 수신
    stompClient.value.subscribe(`/sub/chat/talk/${props.chat.chatSeq}`, (message) => {
      const content = JSON.parse(message.body);
      suggestMessage.value = content.suggestMessage;

      let msgType;
      if (content.message.includes("생성")) {
        msgType = "ENTER";
      } else {
        msgType = currentUserSeq == content.sendSeq ? "SENT" : "RECEIVED";
      }

      // 이미 큐에 들어있는 메세지인지 확인
      const isDuplicate = messages.value.some(
          (msg) => msg.chatMessageSeq === content.chatMessageSeq
      );

      if(!isDuplicate) {
        messages.value.push({
          chatMessageSeq: content.chatMessageSeq,
          chatRoomSeq: props.chatRoomSeq,
          senderNickName: content.senderNickName,
          sendSeq: content.sendSeq,
          receiveSeq: content.receiveSeq,
          text: content.message,
          type: msgType,
          regDate: new Date(),
          readYn: "N"
        });
      }

      scrollToBottom();
    });

    // 종료 메시지 수신 처리
    stompClient.value.subscribe(`/sub/chat/exit/${props.chat.chatSeq}`, (message) => {
      const content = JSON.parse(message.body);

      if (content.type == "END") {
        alert("상대방이 채팅을 종료했습니다.");
        disconnect();
        emit("goBack"); // 뒤로가기 이벤트 실행
      }
    });

    // 방에 입장 메시지 전송
    stompClient.value.send(`/pub/chat/enter/${props.chat.chatSeq}`, {}, JSON.stringify({
      chatMessageSeq: uuidv4(),
      chatRoomSeq: props.chat.chatSeq,
      senderNickName: receiveUsername.value
    }));

  });

  stompClient.value.onclose = () => {
    isConnected.value = false;
  };
}


function sendMessage() {
  if (newMessage.value.trim() && stompClient.value && stompClient.value.connected) {
    const payload = {
      chatMessageSeq: uuidv4(),
      chatRoomSeq: props.chat.chatSeq,
      senderNickName: sendUsername.value,
      sendSeq: currentUserSeq,
      receiveSeq: currentUserSeq==props.chat.sendUserSeq ? props.chat.receiveUserSeq : props.chat.sendUserSeq,
      message: newMessage.value,
      type: "TALK",
      readYn: "N",
      userYn: props.chat.receiveUserSeq==currentUserSeq ? "Y" : "N"
    };

    try {
      stompClient.value.send(`/pub/chat/talk/${props.chat.chatSeq}`, {}, JSON.stringify(payload));
      messages.value.push({
        chatMessageSeq: payload.chatMessageSeq,
        chatRoomSeq: payload.chatRoomSeq,
        senderNickName: payload.senderNickName,
        sendSeq: payload.sendSeq,
        receiveSeq: payload.receiveSeq,
        text: payload.message,
        type: "SENT",
        regDate: new Date(),
        readYn: "N"
      });
      newMessage.value = '';
      scrollToBottom();
    } catch (error) {
      console.error("Error sending message:", error);
    }
  }
}

function disconnect() {
  if (stompClient.value && stompClient.value.connected) {
    stompClient.value.disconnect(() => {
      console.log("WebSocket 연결이 종료되었습니다.");
    });
    isConnected.value = false;
  }
}

async function disconnectEvent() {
  if(confirm("채팅을 종료하시겠습니까?")) {
    try {
      const endMessage = {
        chatMessageSeq: uuidv4(),
        chatRoomSeq: props.chat.chatSeq,
        senderNickName: sendUsername.value,
        sendSeq: currentUserSeq,
        receiveSeq: currentUserSeq==props.chat.sendUserSeq ? props.chat.receiveUserSeq : props.chat.sendUserSeq,
        message: '',
        type: "END",
      };
      stompClient.value.send(`/pub/chat/exit/${props.chat.chatSeq}`, {}, JSON.stringify(endMessage));
      messages.value.push({
        chatMessageSeq: endMessage.chatMessageSeq,
        chatRoomSeq: endMessage.chatRoomSeq,
        senderNickName: endMessage.senderNickName,
        sendSeq: endMessage.sendSeq,
        receiveSeq: endMessage.receiveSeq,
        text: endMessage.message,
        type: "END",
      });

      disconnect();
      emit("goBack");

      alert("채팅이 종료되었습니다.");
    } catch (error) {
      console.error("endDate 변경 중 오류 발생:", error);
      alert("채팅 종료 중 오류가 발생하였습니다. 다시 시도해주세요.");
    }
  }
}

function goEvaluation() {
  if(confirm("평가 화면으로 이동하시겠습니까?")) {
    // 새로운 창에서 경로 열기
    const url = `/chat/${props.chat.chatSeq}/evaluation/`;
    window.open(url, '_blank');
  }
}

function scrollToBottom() {
  nextTick(() => {
    if (messagesContainer.value) {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
    }
  });
}

function formatDate(regDate) {
  return dayjs(regDate).format('YYYY-MM-DD HH:mm'); // 원하는 형식으로 변환
}

function setInputMessage(message) {
  console.log(message);
  newMessage.value = message;
}

</script>

<template>
  <div class="chat-room">
    <button class="chat-room-button back" @click="emit('goBack')"> < </button>

    <button v-if="chat.showEvaluation" class="chat-room-button disconn" id="disconn" @click="goEvaluation"> 평가하기</button>
    <button v-else class="chat-room-button disconn" id="disconn" @click="disconnectEvent">종료하기</button>

    <hr style="border: 1px solid #262627; margin: 10px 0;" />

    <div class="messages" ref="messagesContainer">
      <div
          v-for="message in messages"
          :key="message.chatMessageSeq"
          :class="['message', message.type]"
      >

          <span v-if="message.type == 'ENTER'" class="enter-message">{{ (sendUsername==userNickname ? receiveUsername : sendUsername) + message.text }}</span>

          <template v-else>
            <div class="message-content" :class="{ 'sent-message': message.type == 'SENT' }">
              <img
                  class="profile-image"
                  src="@/images/profile-image.jpg"
                  alt="프로필 이미지"
                  :style="message.type == 'SENT' ? 'margin-left: 10px;' : 'margin-right: 10px;'"
              />
              <div class="message-details">
              <span
                  class="sender"
                  :class="{
                    'sender-sent': message.type == 'SENT',
                    'sender-received': message.type == 'RECEIVED'
                  }"
              >
              {{ message.type == 'SENT' ? userNickname : (sendUsername==userNickname ? receiveUsername : sendUsername) }}
            </span><br />
            <span class="text">{{ message.text }}</span><br />
            <span class="date">{{ formatDate(message.regDate) }}</span><br />
          </div>
        </div>
        </template>
      </div>
    </div>

    <!-- 추천 메시지 -->
    <div v-if="suggestMessage !== null && suggestMessage !== '' && chat.receiveUserSeq==currentUserSeq" class="recommended-messages">
      <button class="recommended-message" @click="setInputMessage(suggestMessage)">
        {{ suggestMessage }}
      </button>
    </div>

    <div class="input-area">
      <input v-if="!chat.showEvaluation" class="sendMessage-area"
          v-model="newMessage"
          @keydown.enter="sendMessage"
          placeholder="메세지를 입력하세요."
          :disabled="!isConnected">
      <input v-else class="sendMessage-area"
             placeholder="종료된 채팅방은 조회만 가능합니다."
             :disabled="true">
      <button v-if="!chat.showEvaluation" class="chat-room-button submit-btn" @click="sendMessage" :disabled="!isConnected"> 전송</button>
      <button v-else class="chat-room-button un-submit-btn" :disabled="true"> 전송</button>
    </div>
  </div>
</template>

<style scoped>
/* 추천 메시지 영역 */
.recommended-messages {
  display: flex;
  justify-content: flex-start; /* 왼쪽 정렬 */
  gap: 10px; /* 버튼 간 간격 */
  padding: 10px 0;
  background-color: #ffffff;
  border-bottom: 1px solid #ddd;
  margin-bottom: 10px;
  max-width: 270px;
  overflow-x: auto; /* 버튼이 많을 때 가로 스크롤 가능 */
}

/* 추천 메시지 버튼 */
.recommended-message {
  cursor: pointer;
  padding: 8px 12px;
  background-color: #e6e6e6;
  border: 1px solid #ccc;
  border-radius: 20px;
  font-size: 14px;
  color: #333;
  white-space: nowrap; /* 텍스트 줄바꿈 방지 */
  transition: background-color 0.2s ease;
}

.recommended-message:hover {
  background-color: #dcdcdc;
}
.message-content {
  display: flex;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.sent-message {
  text-align: right;
  flex-direction: row-reverse;
}

.profile-image {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover; /* 이미지가 비율에 맞게 잘림 */
  border: 1px solid #ddd; /* 테두리 추가 */
}

.message-details {
  max-width: 70%; /* 메시지 내용 너비 제한 */
}

.chat-room-button.back {
  all: unset;
  height: 38px;
  cursor: pointer;
  font-size: 20px;
  font-weight: bold;
}

.chat-room-button.disconn {
  width: 100px;
  height: 38px;
  background-color: #262627;
  color: #ffffff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 15px;
  font-weight: bold;
  float : right;
}

.sendMessage-area {
  height: 40px;
  border-style: solid;
  border-radius: 5px;
}

.chat-room-button.submit-btn {
  width: 70px;
  height: 40px;
  background-color: #CCB997;
  color: #262627;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 15px;
  font-weight: bold;
  float : right;
}
.chat-room-button.un-submit-btn {
  width: 70px;
  height: 40px;
  background-color: lightgrey;
  border: none;
  border-radius: 5px;
  font-size: 15px;
  font-weight: bold;
  float : right;
}

.chat-room {
  max-width: 600px;
  margin: 0 auto;
  padding: 1rem;
}

.messages {
  margin-top: 10px;
  height: 450px;
  overflow-y: auto;
  border: 1px solid #ddd;
  padding: 1rem;
  margin-bottom: 1rem;
}

.message {
  font-size: 12px;
  margin-bottom: 0.5rem;
}

/* 입장 메시지 스타일 (가운데 정렬) */
.message.ENTER {
  align-self: center;
  text-align: center;
  color: #333333;
  font-size: 12px;
  border: 1px solid #ddd;
  border-radius: 15px;
  background-color: lightgrey;
  margin-bottom: 20px;
}

.message.sent {
  align-self: flex-start;
  text-align: right;
  color: black;
}

.message.received {
  align-self: flex-end;
  text-align: left;
  color: black;
}

/* 공통 닉네임 스타일 */
.sender {
  font-weight: bold;
  font-size: 14px;
  margin-bottom: 0.2rem;
}

/* 보낸 사람 닉네임 스타일 */
.sender-sent {
  font-weight: bold;
  color: #555555;
  text-decoration: underline
}

/* 받은 사람 닉네임 스타일 */
.sender-received {
  font-weight: bold;
  color: green; /* 받은 사람은 초록색 */
  text-decoration: underline;
}

/* 날짜 스타일 */
.date {
  font-size: 0.5rem;
  color: #333333;
}

.input-area {
  display: flex;
  gap: 0.5rem;
}

input {
  flex: 1;
  padding: 0.5rem;
}

</style>
