<template>
  <v-main class="wrap">
    <v-container class="text-center">
      <v-row justify="center" align="center">
        <v-col cols="12" md="6">
          <v-text-field :value="gameStatus" readonly></v-text-field>
        </v-col>
      </v-row>
      <v-row justify="center" align="center">
        <v-col cols="12" md="6">
          <p class="message-box">{{ messageToReceived }}</p>
          <v-text-field
            v-model="messageToSend"
            v-if="this.messageToReceived"
            variant="underlined"
          ></v-text-field>
          <v-btn
            color="accent"
            large
            @click.prevent="sendMessage"
            v-if="!messageSent"
            style="margin-top: 5px"
            >메시지 전송</v-btn
          >
          <v-btn
            color="accent"
            large
            @click.prevent="disconnect"
            style="margin-top: 5px; margin-left: 10px"
            >나가기</v-btn
          >
        </v-col>
      </v-row>
    </v-container>
  </v-main>
</template>
<style>
.message-box {
  border: none;
  text-align: center;
  font-size: 1.2em;
  margin-top: 50px;
  margin-bottom: 20px;
  color: #333;
}
.wrap {
  background-image: url("../assets/background.jpg");
  background-size: cover;
  background-position: center;
  width: 100%;
  height: 100%;
}
</style>
<script>
import Stomp from "stompjs";
import SockJS from "sockjs-client";

export default {
  props: ["id"],
  data() {
    return {
      connected: false,
      messageToSend: "",
      messageToReceived: "",
      messageSent: false, // 메시지 전송 여부를 나타내는 속성
    };
  },
  computed: {
    gameStatus() {
      return this.messageToReceived
        ? "전원 입장 완료!"
        : "제한 인원이 모두 입장하면 게임이 시작됩니다.";
    },
    
  },
  created() {
    this.connect();
    window.addEventListener('beforeunload', this.handleBeforeUnload);
  },
  beforeRouteLeave(to, from, next) {
    // 라우트를 벗어나기 전에 웹소켓 연결 해제
    // 게임 입장 후 뒤로가기 누를 경우 disconnect
    this.disconnect();
    next();
  },
  methods: {
    connect() {
      // 웹 소켓 서버 URL
      const serverURL = `${process.env.VUE_APP_API_BASE_URL}/portfolio`;

      // SockJS로 서버에 연결
      let socket = new SockJS(serverURL);

      // Stomp 클라이언트 생성
      this.stompClient = Stomp.over(socket);

      console.log(`소켓 연결을 시도합니다. 서버 주소: ${serverURL}`);

      // Stomp 클라이언트로 서버에 연결
      const token = localStorage.getItem("Authorization");
      const headers = token ? { Authorization: `Bearer ${token}` } : {};
      const destination = `/topic/sharing/${this.id}`;
      this.stompClient.connect(
        headers,
        (frame) => {
          console.log("소켓 연결 성공", frame);
          this.connected = true;

          // 연결될 때마다 백에서 Client count++
          this.stompClient.send(
            "/app/connect",
            {
              destination: "/app/connect",
            },
            JSON.stringify({
              roomNumber: `${this.id}`,
            })
          );

          // 메시지 구독
          this.stompClient.subscribe(destination, (message) => {
            console.log("메시지 수신", message);

            // 전체 입장 완료 시 5초 카운트다운 시작
            if (message) {
              let countdown = 5;
              const countdownInterval = setInterval(() => {
                this.messageToReceived = `게임이 ${countdown}초 뒤 시작합니다`;
                countdown--;
                if (countdown === 0) {
                  clearInterval(countdownInterval);
                }
              }, 1000);
            }

            // 수신한 메시지를 5초 뒤에 messageToReceived 변수에 할당
            setTimeout(() => {
              this.messageToReceived = message.body;
            }, 6000);
          });

          const destination2 = `/queue/sharing/${this.id}`;
          this.stompClient.subscribe(destination2, (message) => {
            console.log("메시지 수신", message);
            const messageIdPrefix = message.headers["message-id"].slice(0, 8);
            const bodyData = JSON.parse(message.body);
            for (const key in bodyData) {
              if (key.startsWith(messageIdPrefix)) {
                this.messageToReceived = bodyData[key];
              }
            }
          });

          const destination_error = `/queue/error/${this.id}`;
          this.stompClient.subscribe(destination_error, (message) => {
            console.log("메시지 수신", message);
            const messageIdPrefix = message.headers["message-id"].slice(0, 8);
            if (message.body === messageIdPrefix) {
              alert("인원이 꽉찼습니다.");
              history.back();
            }
          });
        },
        (error) => {
          console.log("소켓 연결 실패", error);
          this.connected = false;
        }
      );
    },
    disconnect() {
      if (this.stompClient && this.stompClient.connected) {
        // 연결될 때마다 백에서 Client count--
        this.stompClient.send(
          "/app/disconnect",
          {
            destination: "/app/disconnect",
          },
          JSON.stringify({
            roomNumber: `${this.id}`,
          })
        );

        this.stompClient.disconnect(); // Stomp 클라이언트 연결 해제
        console.log("소켓 연결이 해제되었습니다.");
        this.connected = false;
        this.$router.push("/sharingHome");
      } else {
        console.log("소켓 연결이 이미 해제되었거나 없습니다.");
      }
    },
    sendMessage() {
      if (this.connected && this.messageToSend.trim() !== "") {
        // this.messageToSend를 소켓을 통해 전송하는 로직
        console.log("전송할 메시지:", this.messageToSend);
        // WebSocket 클라이언트로 메시지 전송
        this.stompClient.send(
          "/app/sendMessage",
          {
            destination: "/app/sendMessage",
          },
          JSON.stringify({
            token: localStorage.getItem("Authorization"),
            content: this.messageToSend,
            roomId: `${this.id}`,
          })
        );
        // 메시지를 전송한 후, 텍스트 필드를 비우고, messageSent를 true로 설정합니다.
        this.messageToSend = "";
        this.messageSent = true;
      } else {
        console.log("연결되지 않았거나 빈 메시지입니다.");
      }
    },
    handleBeforeUnload(event) {
      this.disconnectFromServer();
      event.returnValue = '';
    },
    disconnectFromServer() {
      if (this.stompClient && this.stompClient.connected) {
        this.stompClient.disconnect();
        console.log("소켓 연결이 해제되었습니다.");
      }
    }
  },
  beforeUnmount() {
    window.removeEventListener('beforeunload', this.handleBeforeUnload);
  }
};
</script>
