<template lang="pug">
  v-layout
    v-container
      v-row
        v-flex.text-center
          div.videos

            video(ref="localVideo" muted autoplay width="240" height="180" )

      v-row
        v-btn(@click="makePeer") make remote peer

      v-row
        v-card(v-for="(conn, index) in conns" :key="index" outlined)
          v-card-text
            v-row
              video(ref="remoteVideo" autoplay width="240" height="180")
              v-btn(@click="createOffer(conn)" v-if="!conn.localSDP && !conn.remoteSDP") create offer
              v-btn(@click="createAnswer(conn)" v-if="!conn.localSDP && conn.remoteSDP") create answer

            v-row
              v-textarea(cols="40" rows="3" :value="conn.localSDP" label="local sdp" outlined readonly)
              v-textarea(cols="40" rows="3" :value="conn.remoteSDP" label="remote sdp" outlined readonly)

            v-row
              v-textarea(cols="40" rows="3" v-if="!conn.remoteSDP" ref="remoteMessage" v-model="conn.remoteMessage" label="paste remote offer here" outlined)
              v-btn(@click="setRemoteSDP(conn)" v-if="!conn.remoteSDP") set remote sdp
            v-row
              v-textarea(cols="40" rows="3" :value="conn.icelog" label="local ice cadidate log" outlined readonly)
              v-textarea(cols="40" rows="3" v-model="conn.remoteIce" label="remote ice cadidate" outlined)
              v-btn(@click="addRemoteIce(conn)" ) add remote ice


</template>

<script>

const videosize = {
  width: 240,
  height: 180
}

//let localStream = null

export default {
  data() {
    return {
      conns: [],
      remoteOffer: [],
      localStream: null
    }
  },
  watch: {
    conns: {
      handler: function(val, old) {
      },
      deep: true
    }
  },
  computed: {
    roomID: function () {
      return this.$route.params.id
    },
    connections: function() {
      return this.$store.state.room.connections
    }
  },
  mounted() {
    console.log(this.roomID)
    var self = this
    this.startLocalMedia()
    .then(function(stream) {
      self.$refs.localVideo.srcObject = stream
      self.localStream = stream
    })

  },
  methods: {
    startLocalMedia() {
      const self = this
      return navigator.mediaDevices.getUserMedia(
          {video: videosize, audio: true}
        )
        .catch(err => console.error(err))
    },

    // start point

    makePeer() {
      let connection = {
        peer: null,
        icelog: '',
      }

      try {
        let peer = new RTCPeerConnection({sdpSemantics:'plan-b'})
        connection.peer = peer
        let self = this
        const iceSeparator = "\n--------\n"
        const iceEnd = "end-of-candidates\n"
        // send any ice candidates to the other peer
        peer.onicecandidate = function (event) {
          if (event.candidate) {
            console.log('on candidate');
            console.log(event.candidate)
            let log = JSON.stringify(event.candidate)
            connection.icelog += log + iceSeparator

          } else {
            console.log("candidate end. ------- phase=" + event.eventPhase);
            connection.icelog += iceEnd
          }
        }
        peer.ontrack = function (e) {
          self.onRemoteTrack(connection, e)
        }

        this.localStream.getTracks().forEach(track => {
            peer.addTrack(track)
        })
      } catch (e) {
        connection.error = e
        console.log('failed to create peer connection')
        console.error(e)
      }
      this.conns.push(connection)
    },

    // controll flow of calling remote

    createOffer(connection) {
      console.log(connection)
      let self = this
      connection.peer.createOffer({
        offerToReceiveAudio: 1,
        offerToReceiveVideo: 1
      })
        .then(function (sdp) {
          connection.peer.setLocalDescription(sdp)

          connection.localSDP = JSON.stringify(sdp)

          self.conns.splice(self.conns.indexOf(connection), 1, connection)
        })
        .catch(e => console.log)
    },

    // handle local <= remote

    setRemoteSDP(connection) {
      connection.remoteSDP = connection.remoteMessage
      let remoteSDP = JSON.parse(connection.remoteMessage)
      connection.peer.setRemoteDescription(remoteSDP)
      let index = this.conns.indexOf(connection)
      this.conns.splice(index, 1, connection)
    },

    // controll flow of answer from remote

    async createAnswer(connection) {
      let index = this.conns.indexOf(connection)
      try {
        const answer = await connection.peer.createAnswer()
        connection.peer.setLocalDescription(answer)
        connection.localSDP = JSON.stringify(answer)
        this.conns.splice(index, 1, connection)
      } catch(e) {
        console.log(e)
      }
    },

    addRemoteIce(connection) {
      const ice = JSON.parse(connection.remoteIce)
      connection.peer.addIceCandidate(ice)
        .then(function() {
          console.log('remote ice was added')
          connection.remoteIce = null
        })
        .catch(function(e){
          console.log('failed to add remote ice')
          console.log(e)
        })
    },

    // webrtc peer connection event

    onRemoteTrack(connection, e) {
      console.log('ontrack')
      console.log(e)

      if (connection.stream != e.streams[0]) {
        connection.stream = e.streams[0]
        const index = this.conns.indexOf(connection)
        const video = this.$refs.remoteVideo[index]
        video.srcObject = connection.stream
        //video.play()
        //this.conns.splice(index, 1, connection)
      }
    }
  }
}
</script>



<style>
video {
  width: 240px;
  height: 180px;
  border: 1px solid black;
}
</style>
