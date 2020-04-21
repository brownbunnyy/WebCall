<template>
    <v-content>
      <v-container fluid>
        <v-row>
          <v-col>
            <v-chip
              class="ma-2"
              color="teal"
              text-color="white"
              v-if="room"
            >
              <v-avatar left>
                <v-icon>mdi-checkbox-marked-circle</v-icon>
              </v-avatar>
              接続中:{{room.name}}
            </v-chip>
          </v-col>
        </v-row>
        <v-row>
          <v-col>
            <template v-for="(s,i) in streams">
              <video
                :key="i"
                :id="s.peerId"
                :srcObject.prop="s.src"
                :muted="s.muted"
                autoplay
                playsinline
                width="40%"
              ></video>
            </template>
          </v-col>
        </v-row>
        <v-row>
          <v-col>
            <v-select 
              @change="connectLocalCamera"
              v-model="selectedAudio"
              :items="audios"
              item-value="value"
              label="マイク"
            />
            <v-select 
              @change="connectLocalCamera"
              v-model="selectedVideo"
              :items="videos"
              item-value="value"
              label="カメラ"
            />
          </v-col>
        </v-row>
        <v-row>
          <v-col>
            <v-text-field v-model="roomId" placeholder="Room ID"/>
            <button @click="leaveRoom" class="button--green">leave</button>
            <button @click="makeRoom" class="button--green">make</button>
          </v-col>
        </v-row>
        <v-textarea
          outlined
          disabled
          v-model="messages"
        ></v-textarea>
        <v-text-field v-model="localText" placeholder="メッセージを送信"/>
        <button @click="sendMessage" class="button--green">send</button>

    </v-container>
  </v-content>
</template>

<script>

export default {
  data () {
    return {
      APIKey: process.env.API_KEY,
      selectedAudio: '',
      selectedVideo: '',
      audios: [],
      videos: [],
      localStream: null,
      streams: [],
      room: null,
      peerId: '',
      roomId: 'test',
      messages: '',
      localText: '',
    }
  },
  computed: {
    now: function (){
      let dd = new Date();
      return `${dd.getMonth()}/${dd.getDate()} ${dd.getHours()}:${dd.getMinutes()}:${dd.getSeconds()}`;
    }
  },
  methods: {
    // changeDevice: function () {
    //   if(this.selectedAudio != '' && this.selectedVideo != ''){
    //     this.connectLocalCamera();
    //   }
    // },
    connectLocalCamera: async function(){
      if(this.selectedAudio == '' && this.selectedVideo == ''){
        return;
      }

      const constraints = {
        audio: this.selectedAudio ? { deviceId: { exact: this.selectedAudio } } : false,
        video: this.selectedVideo ? { deviceId: { exact: this.selectedVideo } } : false
      }
      constraints.video.width = {
        min: 320,
        max: 320
      };
      constraints.video.height = {
        min: 240,
        max: 240
      };
      this.localStream = await navigator.mediaDevices.getUserMedia(constraints);
      this.addStream(this.localStream,true);
    },

    // ルームから退出
    leaveRoom: function () {
      if (this.room) {
        this.room.close();
        this.room = null;
      }
    },

    // ルームに参加
    makeRoom: function () {
      this.connectLocalCamera().then(()=>{
        const room = this.peer.joinRoom(this.roomId, {
          mode: 'sfu',
          stream: this.localStream,
        });
        this.connect(room);
      });
    },

    connect: function (room) {
      if (this.room) {
        this.room.close();
      }
      this.room = room;

      room.once('open', () => {
        this.messages += this.now + '=== You joined ===\n';
      });

      room.on('peerJoin', peerId => {
        this.messages += this.now +`=== ${peerId} joined ===\n`;
      });

      room.on('peerLeave', peerId => {
          this.messages += this.now +`=== ${peerId} lefted ===\n`;
          this.removeStream(peerId);
      });

      room.on('data', ({ data, src }) => {
        this.messages += this.now + `${src}: ${data}\n`;
      });

      room.once('close', () => {
          this.messages += this.now +`=== You lefted ===\n`;
          this.removeStream(this.peerId);
      });

      room.on('stream', stream => {
          this.addStream(stream);
      });
    },

    addStream: function (stream, is_local = false) {
      let peerId = is_local ? this.peerId : stream.peerId;
      let is_exist = this.streams.find(st => st.peerId == peerId);
      if (is_exist) {
        return;
      }
      this.streams.push({
        peerId: peerId,
        src: stream,
        muted: is_local,
      });
    },

    removeStream: function (peerId) {
      let index = this.streams.findIndex( stream => {
        return stream.peerId === peerId
      });
      this.streams[index].src.getTracks().forEach(track => track.stop());
      this.streams[index].src = null;
      this.streams.splice(index);
    },

    sendMessage: function () {
      if (!this.room) {
        alert('ルームに参加していません');
        return;
      }
      this.room.send(this.localText);
      this.messages += this.now + `${this.peerId}: ${this.localText}\n`;
      this.localText = '';
    },
  },

  mounted: function () {
    const Peer = require("skyway-js");
    this.peer = new Peer({
      key:   this.APIKey,
      debug: 3,
    });

    this.peer.on('open', () => {
      this.peerId = this.peer.id;
    });

    //デバイスへのアクセス
    navigator.mediaDevices.enumerateDevices()
    .then((deviceInfos) => {
      for (let i = 0; i !== deviceInfos.length; ++i) {
        const deviceInfo = deviceInfos[i]
        if (deviceInfo.kind === 'audioinput') {
          this.audios.push({
            text: deviceInfo.label ||
            `Microphone ${this.audios.length + 1}`,
            value: deviceInfo.deviceId
          })
        } else if (deviceInfo.kind === 'videoinput') {
          this.videos.push({
            text: deviceInfo.label ||
            `Camera  ${this.videos.length - 1}`,
            value: deviceInfo.deviceId
          })
        }
      }
    }).then(()=>{
      this.selectedAudio = this.audios[0].value;
      this.selectedVideo = this.videos[0].value;
    });
  }
}
</script>
