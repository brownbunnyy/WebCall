<template>
<v-app>
    <v-navigation-drawer :clipped="$vuetify.breakpoint.lgAndUp" v-model="dialog.drawer" app>
        <v-card class="mx-auto">
            <v-list three-line>
                <template v-for="(message,index) in messages">
                    <v-list-item :key="index">
                        <v-list-item-avatar>
                            <v-avatar :color="message.color" size="20"></v-avatar>
                        </v-list-item-avatar>
                        <v-list-item-content>
                            <v-list-item-title v-html="message.peerId"></v-list-item-title>
                            <v-list-item-subtitle v-html="message.time"></v-list-item-subtitle>
                            <v-list-item-subtitle v-html="message.value"></v-list-item-subtitle>
                        </v-list-item-content>
                    </v-list-item>
                </template>
            </v-list>
        </v-card>
    </v-navigation-drawer>

    <!-- app-bar -->
    <v-app-bar app dense :clipped-left="$vuetify.breakpoint.lgAndUp" color="blue-grey">
        <v-app-bar-nav-icon @click.stop="dialog.drawer = !dialog.drawer" />
        <v-toolbar-title>WebCall</v-toolbar-title>
        <v-spacer></v-spacer>
        <!-- ルームダイアログ -->
        <div class="text-center">
            <v-dialog v-model="dialog.room">
                <!-- 接続中アイコン -->
                <template v-slot:activator="{ on }">
                    <!-- 接続中アイコン -->
                    <v-chip @click="leaveRoom()" class="ma-2" color="orange" text-color="white" v-if="room">
                        <v-avatar left>
                            <v-icon>mdi-checkbox-marked-circle</v-icon>
                        </v-avatar>
                        接続中：{{room.name}}（{{streams.length}}）
                    </v-chip>
                    <!-- /接続中アイコン -->
                    <!-- 接続アイコン -->
                    <v-chip class="ma-2" color="green" text-color="white" v-else v-on="on">
                        <v-avatar left>
                            <v-icon>mdi-plus-circle</v-icon>
                        </v-avatar>
                        接続
                    </v-chip>
                    <!-- /接続アイコン -->
                </template>
                <v-card>
                    <v-card-title>
                        Join or Make Room
                    </v-card-title>
                    <v-card-text>
                        <v-row>
                            <v-col>
                                <v-text-field v-model="roomId" placeholder="Room ID" label="ルームID" />
                            </v-col>
                        </v-row>
                    </v-card-text>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-row>
                            <v-col>
                                <button @click="makeRoom()" class="button--green">Join or Make</button>
                            </v-col>
                        </v-row>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </div>
        <!-- /ルームダイアログ -->

        <!-- コンフィグダイアログ -->
        <!-- <div class="text-center">
            <v-dialog v-model="dialog.config">
                <template v-slot:activator="{ on }">
                    <v-icon :disabled="!localStream" v-on="on">mdi-cog</v-icon>
                </template>
                <v-card>
                    <v-card-title>
                        <v-icon>mdi-camera</v-icon>
                        <v-icon>mdi-microphone</v-icon>
                    </v-card-title>
                    <v-card-text>
                        <v-row>
                            <v-col>
                                <v-select @change="changeDevice" v-model="selectedAudio" :items="audios" item-value="value" label="マイク" />
                                <v-select @change="changeDevice" v-model="selectedVideo" :items="videos" item-value="value" label="カメラ" />
                            </v-col>
                        </v-row>
                    </v-card-text>
                </v-card>
            </v-dialog>
        </div> -->
        <!-- /コンフィグダイアログ -->
    </v-app-bar>
    <!-- /app-bar -->
    <v-content>
        <v-container class="fill-height" fluid>
            <v-row justify="center" align-content="center" v-if="streams.length">
                <v-col cols="auto" v-for="(s,i) in streams" :key="i">
                    <v-card shaped>
                        <v-container>
                            <v-row justify="space-between">
                                <v-col>
                                    <v-card-text>
                                        <video :id="s.peerId" :srcObject.prop="s.src" :muted="true" autoplay playsinline></video>
                                    </v-card-text>
                                    <v-card-title class="headline" v-text="s.peerId"></v-card-title>
                                </v-col>
                            </v-row>
                        </v-container>
                    </v-card>
                </v-col>
            </v-row>
            <v-row align="center" justify="center" v-else>
                <v-col cols="auto">
                    <v-card>
                        <v-card-title>How to get started</v-card-title>
                        <v-card-text>
                            <v-img width="300" src="/start.png"></v-img>
                        </v-card-text>
                    </v-card>
                </v-col>
            </v-row>
        </v-container>
    </v-content>

    <v-btn bottom color="pink" dark fab fixed right @click="dialog.sheet = !dialog.sheet">
        <v-icon>mdi-chat</v-icon>
    </v-btn>
    <div class="text-center">
        <v-bottom-sheet v-model="dialog.sheet">
            <v-sheet class="text-center" height="250px">
                <v-btn class="mt-6" text color="red" @click="dialog.sheet = !dialog.sheet">close</v-btn>
                <v-container>
                    <v-textarea rows="3" outlined v-model="localText" placeholder="メッセージを送信" />
                    <button @click="sendMessage" class="button--green">send</button>
                </v-container>
            </v-sheet>
        </v-bottom-sheet>
    </div>
    <v-snackbar top :timeout="1500" v-model="snackbar">
        {{snackbarMessage}}
    </v-snackbar>
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
            roomId: Math.random().toString(32).substring(2),
            messages: [],
            localText: '',

            dialog: {
                room: false,
                config: false,
                drawer: true,
                sheet: false,
            },
            constraints: {
                audio: true,
                video: {
                    height: {
                        exact: 400
                    },
                    width: {
                        exact: 300
                    },
                }
            },
            snackbar: false,
            snackbarMessage: '',
        }
    },
    computed: {
        now: function () {
            let dd = new Date();
            return `${dd.getMonth()}/${dd.getDate()} ${dd.getHours()}:${dd.getMinutes()}:${dd.getSeconds()}`;
        }
    },
    methods: {
        connectDevice: async function () {

            if (this.localStream) {
                this.localStream = null;
                this.removeStream(this.peerId);
            }

            await navigator.mediaDevices.getUserMedia(this.constraints)
                .then((stream) => {
                    this.localStream = stream;
                    this.addStream(this.localStream, true);
                    return true;
                })
                .catch(err => {
                    alert(err);
                    return false;
                });
        },

        getDevices: function () {
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
            this.connectDevice().then(() => {
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
                this.localStream = null;
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

            this.dialog.room = false;
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
            if (!~index) {
                return
            }
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
            this.dialog.drawer = true;
            this.snackbar = true;
            this.snackbarMessage = `${peerId}: ${message}`;
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
        this.getDevices();
        this.pushMessage('system', 'Welcome!');
    }
}
</script>
