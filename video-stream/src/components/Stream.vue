<template>
  <div>
    <el-container>
      <el-main>
        <el-row>
          <el-button type="succes" @click="login">Login</el-button>
          <h1>Ứng dụng VideoStream</h1>
          <div>
              <el-button type="primary" @click="createRoom">Tạo meeting</el-button>
              <el-button type="danger" @click="joinWithId">Join meeting</el-button>
              <el-button type="warning" @click="publish(true)">Share desktop</el-button>
          </div>
          <div>Bạn đang ở trong room {{roomId}} </div>
          <div>Gửi link cho bạn bè {{roomToken}}</div>
        </el-row>
        <div class="container">
          <div id="videos"></div>
        </div>
      </el-main>
    </el-container>
    
  </div>
</template>

<script>
import {api} from '../api.js'


export default {
  
  data() {
     return {
       userToken: "",
       roomId: "",
       roomToken: "",
       room : undefined,
       client: undefined,
     }
  },
  created() {
    console.log('window', window)
  },
  methods: {
    login() {
      return new Promise(resolve => {
        const userId = (Math.random()*10000).toFixed(0)
        api.getUserToken(userId)
        .then((utk) => {
          const client = new window.StringeeClient();
          client.on('authen', (res) => {
            console.log('on authen' , res)
            resolve(res);
          })
          // đoạn này lấy api xong rồi mới connect
          client.connect(utk)
          this.client = client  
        })
      })
    },
    async publishVideo() {
      const localTrack = await window.StringeeVideo.createLocalVideoTrack(this.client, {
        audio: true,
        video: true,
        videoDimensions: {width: 640, height: 360}
      })
      const videoContainer = document.querySelector("#videos")
      const videoElement = localTrack.attach()
      videoContainer.appendChild(videoElement)

      const roomData = await window.StringeeVideo.joinRoom(this.client, this.roomToken)
      const room = roomData.room

      console.log({room, roomData})
      this.room = room

      room.clearAllOnMethos();
      room.on('addtrack', ( async(event) => {
        const trackInfo = event.info.track

        if (trackInfo.serverId === localTrack.serverId){
          return
        }
        const track = await room.subscribe(trackInfo.serverId);

        track.on('ready', () =>{
          const ele = track.attach();
          videoContainer.appendChild(ele)
        })
      }))

      room.on('removetrack', (event) => {
        if(!event.track) {
          return
        }
        const elements = event.track.detach()
        elements.forEach(element => element.remove())
      })

      roomData.listTracksInfo.forEach(async(trackInfo) => {
          const track = await room.subscribe(trackInfo.serverId);
          track.on('ready', () =>{
          const ele = track.attach();
          videoContainer.appendChild(ele)
        })
      })

      room.publish(localTrack)
      
    },
    async createRoom() {
      const room =  await api.createRoom()
      const roomToken = await api.getRoomToken(room.roomId)
      this.roomId = room.roomId
      this.roomToken = roomToken

      await this.login()
      await this.publishVideo()
    },
    async joinWithId() {
      const roomId = prompt('Dán roomId vào đây')
      if(!roomId) {
        return
      }
      const roomToken = await api.getRoomToken(roomId)
      this.roomId = roomId
      this.roomToken = roomToken
      
      this.login();
      this.publishVideo();
    },

  
    publish() {

    }
  },
  computed: {

  },
  mounted() {
    api.setRestToken()
  }
}
</script>