<template>
    <v-content>
      <v-container fluid>
        <v-row>
          <v-col>
            <h2>room: <span id="room-name" v-if="room">{{room.name}}</span></h2>
            <template v-for="(stream,i) in remoteStreams">
              <video :key="i" :id="stream.id" :srcObject.prop="stream.src" width="50%" autoplay></video>
            </template>
          </v-col>
        </v-row>
        <v-row>
          <v-col>
            <v-select 
              @change="changeDevice"
              v-model="selectedAudio"
              :items="audios"
              item-value="value"
              label="マイク"
            />
            <v-select 
              @change="changeDevice"
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
      remoteStreams: [],
      room: null,
      peerId: '',
      roomId: 'test',
      messages: '',
      localText: '',
    }
  },
  methods: {
    changeDevice: function () {
      if(this.selectedAudio != '' && this.selectedVideo != ''){
        this.connectLocalCamera();
      }
    },

    connectLocalCamera: async function(){
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
      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      this.localStream = stream;
      this.localStream.peerId = this.peerId;
      this.addStream(this.localStream);
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
      this.changeDevice();
      const room = this.peer.joinRoom(this.roomId, {
        mode: 'mesh',
        stream: this.localStream,
      });
      this.connect(room);
    },

    connect: function (room) {
      if (this.room) {
        this.room.close();
      }
      this.room = room;

      room.once('open', () => {
        this.messages += (new Date()) + '=== You joined ===\n';
      });

      room.on('peerJoin', peerId => {
        this.messages += (new Date()) +`=== ${peerId} joined ===\n`;
      });

      room.on('peerLeave', peerId => {
          this.messages += (new Date()) +`=== ${peerId} lefted ===\n`;
          this.removeStream(peerId);
      });

      room.on('data', ({ data, src }) => {
        this.messages += (new Date()) + `${src}: ${data}\n`;
      });

      room.once('close', () => {
          this.messages += (new Date()) +`=== You lefted ===\n`;
          this.removeStream(this.room.remoteId);
      });

      room.on('stream', stream => {
        this.addStream(stream);
      });
    },

    addStream: function (stream) {
      let peerId = (stream.peerId) ? stream.peerId : this.peerId;
      let is_exist = this.remoteStreams.find( stream => {
        return stream.src.peerId == peerId
      });
      if (is_exist) {
        return;
      }
      this.remoteStreams.push({
        id: 'data-peer-id-' + peerId,
        src: stream,
      })
    },

    removeStream: function (peerId) {
      let index = this.remoteStreams.findIndex( stream => {
        return stream.peerId === peerId
      });
      this.remoteStreams[index].src.getTracks().forEach(track => track.stop());
      this.remoteStreams[index].src = null;
      this.remoteStreams.splice(index);
    },

    sendMessage: function () {
      if (!this.room) {
        alert('ルームに参加していません');
        return;
      }
      this.room.send(this.localText);
      this.messages += (new Date()) + `${this.peerId}: ${this.localText}\n`;
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
