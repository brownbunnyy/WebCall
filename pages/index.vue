<template>
<v-app>
    <!-- app-bar -->
    <v-app-bar app dense color="blue">
        <v-toolbar-title>Web Call</v-toolbar-title>
        <v-spacer></v-spacer>
        <v-chip class="ma-2" color="teal" text-color="white" v-if="room">
            <v-avatar left>
                <v-icon>mdi-checkbox-marked-circle</v-icon>
            </v-avatar>
            接続中：{{room.name}}（{{streams.length}}）
        </v-chip>
        <v-chip class="ma-2" color="orange" text-color="white" v-else>
            <v-avatar left>
                <v-icon>mdi-close-circle</v-icon>
            </v-avatar>
            未接続
        </v-chip>

        <!-- dialog -->
        <div class="text-center">
            <v-dialog v-model="dialog">
                <template v-slot:activator="{ on }">
                    <v-btn color="red lighten-2" dark v-on="on">
                        Config
                    </v-btn>
                </template>
                <v-card>
                    <v-card-title class="headline grey lighten-2" primary-title>
                        Camera ＆ Microphone setting
                    </v-card-title>
                    <v-card-text>
                        <v-row>
                            <v-col>
                                <v-select @change="connectLocalCamera" v-model="selectedAudio" :items="audios" item-value="value" label="マイク" />
                                <v-select @change="connectLocalCamera" v-model="selectedVideo" :items="videos" item-value="value" label="カメラ" />
                                <v-text-field v-model="roomId" placeholder="Room ID" label="ルームID" />
                            </v-col>
                        </v-row>
                    </v-card-text>
                    <v-divider></v-divider>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-row>
                            <v-col>
                                <button @click="leaveRoom();dialog = false" class="button--green">leave</button>
                                <button @click="makeRoom();dialog =false" class="button--green">make</button>
                            </v-col>
                        </v-row>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </div>
        <!-- dialog -->

    </v-app-bar>
    <!-- app-bar -->
    <v-content>
        <v-container fluid fill-heigh>
            <v-row justify="center" align-content="center" v-if="streams.length">
                <v-col cols="auto" v-for="(s,i) in streams" :key="i">
                    <v-card shaped hover width="340" height="350">
                        <v-container>
                            <v-row justify="space-between">
                                <v-col cols="auto">
                                    <video :id="s.peerId" :srcObject.prop="s.src" :muted="true" autoplay playsinline width="320" height="240"></video>
                                    <v-card-title class="headline" v-text="s.peerId"></v-card-title>
                                </v-col>
                            </v-row>
                        </v-container>
                    </v-card>
                </v-col>
            </v-row>
            <v-row style="height: 300px;" justify="center" align-content="center" v-else>
                <v-col cols="auto">
                    <v-card shaped hover class="mx-auto">
                        <v-container>
                            <v-row justify="space-between">
                                <v-col cols="auto">
                                    <v-avatar color="indigo">
                                        <v-icon dark>mdi-account-circle</v-icon>
                                    </v-avatar>
                                    <v-card-title class="headline">No one else...</v-card-title>
                                </v-col>
                            </v-row>
                        </v-container>
                    </v-card>
                </v-col>
            </v-row>
            <v-row>
                <v-col>
                    <v-card fluid class="mx-auto">
                        <v-card-title class="blue-grey white--text" dense>
                            <span class="title">chat</span>
                        </v-card-title>
                        <v-card-text v-if="messages.length">
                            <v-timeline dense>
                                <v-timeline-item v-for="(message,index) in messages" :key="index" :color="message.color" small fill-dot>
                                    <v-card class="elevation-2">
                                        <v-card-title class="headline">{{message.peerId}}</v-card-title>
                                        <v-card-subtitle>{{message.time}}</v-card-subtitle>
                                        <v-card-text>{{message.value}}</v-card-text>
                                    </v-card>
                                </v-timeline-item>
                            </v-timeline>
                        </v-card-text>
                        <v-card-actions>
                            <v-row>
                                <v-col cols="11">
                                    <v-text-field outlined v-model="localText" placeholder="メッセージを送信" />
                                </v-col>
                                <v-col cols="1">
                                    <button @click="sendMessage" class="button--green">send</button>
                                </v-col>
                            </v-row>
                        </v-card-actions>
                    </v-card>
                </v-col>
            </v-row>
        </v-container>
    </v-content>
</v-app>
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

            dialog: false,
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
