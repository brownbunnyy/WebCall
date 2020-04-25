<template>
<v-content>
    <v-container fluid>
        <v-row>
            <v-col>
                <v-chip class="ma-2" color="teal" text-color="white" v-if="room">
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
                    <video :key="i" :id="s.peerId" :srcObject.prop="s.src" :muted="s.muted" autoplay playsinline width="40%"></video>
                </template>
            </v-col>
        </v-row>
        <v-row>
            <v-col>
                <v-select @change="connectLocalCamera" v-model="selectedAudio" :items="audios" item-value="value" label="マイク" />
                <v-select @change="connectLocalCamera" v-model="selectedVideo" :items="videos" item-value="value" label="カメラ" />
            </v-col>
        </v-row>
        <v-row>
            <v-col>
                <v-text-field v-model="roomId" placeholder="Room ID" />
                <button @click="leaveRoom" class="button--green">leave</button>
                <button @click="makeRoom" class="button--green">make</button>
            </v-col>
        </v-row>
        <v-row>
            <v-col>
                <v-card class="mx-auto">
                    <v-card-title class="blue-grey white--text">
                        <span class="title">Logs</span>
                    </v-card-title>
                    <v-card-text class="py-0">
                        <v-timeline dense>
                            <v-slide-x-reverse-transition group hide-on-leave>
                                <v-timeline-item v-for="(message,index) in messages" :key="index" color="info" small fill-dot>
                                    <template v-slot:icon>
                                        <v-avatar :color="message.color" size="36">
                                            <span class="white--text headline"></span>
                                        </v-avatar>
                                    </template>
                                    <v-card class="elevation-2">
                                        <v-card-title class="headline">{{message.peerId}}</v-card-title>
                                        <v-card-subtitle>{{message.time}}</v-card-subtitle>
                                        <v-card-text>
                                            {{message.value}}
                                        </v-card-text>
                                    </v-card>
                                </v-timeline-item>
                            </v-slide-x-reverse-transition>
                        </v-timeline>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
        <v-row>
            <v-col>
                <v-textarea outlined v-model="localText" placeholder="メッセージを送信" />
                <button @click="sendMessage" class="button--green">send</button>
            </v-col>
        </v-row>
    </v-container>
</v-content>
</template>

<script>
export default {
    data() {
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
            messages: [],
            localText: '',
        }
    },
    computed: {
        now: function () {
            let dd = new Date();
            return `${dd.getMonth()}/${dd.getDate()} ${dd.getHours()}:${dd.getMinutes()}:${dd.getSeconds()}`;
        }
    },
    methods: {
        connectLocalCamera: async function () {
            if (this.selectedAudio == '' && this.selectedVideo == '') {
                return;
            }

            const constraints = {
                audio: this.selectedAudio ? {
                    deviceId: {
                        exact: this.selectedAudio
                    }
                } : false,
                video: this.selectedVideo ? {
                    deviceId: {
                        exact: this.selectedVideo
                    }
                } : false
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
            this.addStream(this.localStream, true);
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
            this.connectLocalCamera().then(() => {
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
                this.pushMessage(this.peerId, 'You joined');
            });

            room.once('close', () => {
                this.pushMessage(this.peerId, 'You lefted');
                this.removeStream(this.peerId);
            });

            room.on('peerJoin', peerId => {
                this.pushMessage(peerId, `${peerId} joined`);
            });

            room.on('peerLeave', peerId => {
                this.pushMessage(peerId, `${peerId} lefted`);
                this.removeStream(peerId);
            });

            room.on('data', ({
                data,
                src
            }) => {
                this.pushMessage(src, data);
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
            let index = this.streams.findIndex(st => {
                return st.peerId === peerId
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
            this.pushMessage(this.peerId, this.localText);
            this.localText = '';
        },
        pushMessage: function (peerId, message) {
            console.log('message');
            this.messages.push({
                peerId: peerId,
                color: peerId.toRGBCode(),
                time: this.now,
                value: message,
            })
        },
    },

    mounted: function () {
        const Peer = require("skyway-js");
        this.peer = new Peer({
            key: this.APIKey,
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
            }).then(() => {
                this.selectedAudio = this.audios[0].value;
                this.selectedVideo = this.videos[0].value;
            });
    }
}
</script>
