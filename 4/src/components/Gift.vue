<template>
  <div id="countdown">  
    <div id='tiles' class="color-full">{{day}}:{{hour}}:{{minute}}:{{second}}</div>
    <div class="countdown-label" id="countText">{{text}}</div>
    <div class="countdown-label" id="posGift" v-for="item in items" v-bind:key="item.giftName">
      一个{{ item.giftName }}{{ item.giftTypeText }}{{ item.giftTime }}分钟
    </div>
  </div>
</template>

<script>
import axios from 'axios'

const COMMAND_HEARTBEAT = 0
const COMMAND_JOIN_ROOM = 1
const COMMAND_ADD_GIFT = 3

export default {
  name: 'Gift',
  data() {
    return {
      timestampEnd:0,
      days: 0,
      hours: 0,
      minutes: 0,
      seconds: 0,
      text: "",
      items: [],
      websocket: null,
      retryCount: 0,
      isDestroying: false,
      config: {},
    }
  },
  created() {
    this.processToken()
  },
  beforeDestroy() {
    this.isDestroying = true
    this.websocket.close()
  },
  watch: {
    second: {
      handler () {
      }
    },
    minute: {
      handler () {
      }
    },
    hour: {
      handler () {
      }
    },
    day: {
      handler () {
      }
    }
  },
  computed: {
    second() {
      return this.seconds
    },
    minute() {
      return this.minutes
    },
    hour() {
      return this.hours
    },
    day() {
      return this.days
    }
  },
  methods: {
    refreshTimer(){
      var lefttime = parseInt((this.timestampEnd - new Date().getTime()) / 1000)
      if(lefttime >= 0){
        var d = parseInt(lefttime / (24 * 60 * 60 ))
        var h = parseInt(lefttime / (60 * 60) % 24)
        var m = parseInt(lefttime / 60 % 60)
        var s = parseInt(lefttime % 60)
        h = this.addZero(h)
        m = this.addZero(m)
        s = this.addZero(s)
        d = this.addZero(d)
        this.hours = h
        this.minutes = m
        this.seconds = s
        this.days = d
      }else{
        this.hours = this.addZero(0)
        this.minutes = this.addZero(0)
        this.seconds = this.addZero(0)
        this.days = this.addZero(0)
      }
    },
    addZero(i){
      return i < 10 ? "0" + i: i + "";
    },
    async processToken(){
      const url = `https://acmate.loli.ren/api/query?token=` + this.$route.params.token
      var data = (await axios.get(url)).data
      if(data.result == 1){
        for (let i = 0; i < data.data.gifts.length; i++) {
          const element = data.data.gifts[i];
          var giftTypeText = "加"
          if(element.gift_type == "-"){
            giftTypeText = "减"
          }
          this.items.push({
            'giftName': element.gift_name,
            'giftType': element.gift_type,
            'giftTypeText': giftTypeText,
            'giftTime': element.gift_amount,
          })
        }
        this.timestampEnd = new Date().getTime() + data.data.liveTime * 1000 * 60 * 60
        this.text = data.data.liveText
        this.config.roomId = data.data.roomID
        this.wsConnect()
        window.setInterval(this.refreshTimer, 1 * 1000)
      }
    },
    wsConnect() {
      const url = `wss://danmaku.loli.ren/chat`
      this.websocket = new WebSocket(url)
      this.websocket.onopen = this.onWsOpen
      this.websocket.onclose = this.onWsClose
      this.websocket.onmessage = this.onWsMessage
      this.heartbeatTimerId = window.setInterval(this.sendHeartbeat, 10 * 1000)
    },
    sendHeartbeat() {
      this.websocket.send(JSON.stringify({
        cmd: COMMAND_HEARTBEAT
      }))
    },
    onWsOpen() {
      this.retryCount = 0
      this.websocket.send(JSON.stringify({
        cmd: COMMAND_JOIN_ROOM,
        data: {
          roomId: parseInt(this.config.roomId),
          version: "9.9.9",
          config: {
            autoTranslate: false
          }
        }
      }))
    },
    onWsClose() {
      if (this.heartbeatTimerId) {
        window.clearInterval(this.heartbeatTimerId)
        this.heartbeatTimerId = null
      }
      if (this.isDestroying) {
        return
      }
      window.console.log(`掉线重连中${++this.retryCount}`)
      this.wsConnect()
    },
    onWsMessage(event) {
      let {cmd, data} = JSON.parse(event.data)
      switch (cmd) {
        case COMMAND_ADD_GIFT:
          for (let i = 0; i < this.items.length; i++) {
            const element = this.items[i];
            if(data.giftName === element.giftName){
              if(element.giftType == "-"){
                this.timestampEnd -= data.num * element.giftTime * 60 * 1000
              }else{
                this.timestampEnd += data.num * element.giftTime * 60 * 1000
              }
            }
          }
          break
      }
    }
  }
}
</script>

<style type="text/css">
      
:root{
  --textColor:#c06;
}

body{ 
    font: normal 26px/26px Arial, Helvetica, sans-serif; 
    word-wrap:break-word;
    background: transparent;
    padding-top: 2em;
}

.countdown-label {
    text-align: center;
    display: block;
    color: #000000;
    margin-top: 0.1em;
    padding-bottom: 0.1em;
}
#countdown{
    box-shadow: 0 1px 2px 0 rgba(1, 1, 1, 0.4);
    width: 300px;
    height: auto;
    text-align: center;
    background: #f1f1f1;
    border-radius: 5px;
    margin: auto;
}

#countdown #tiles{
  color: #fff;
  position: relative;
  z-index: 1;
  text-shadow: 1px 1px 0px #ccc;
  display: inline-block;
  font-family: Arial, sans-serif;
  text-align: center;
  
  padding: 20px;
  border-radius: 5px 5px 0 0;
  font-size: 48px;
  font-weight: thin;
  display: block;
    
}

.color-full {
  background: #53bb74;
}
.color-half {
  background: #ebc85d;
}
.color-empty {
  background: #e5554e;
}

#countdown #tiles > span{
    width: 70px;
    max-width: 70px;

    padding: 18px 0;
    position: relative;
}





#countdown .labels{
    width: 100%;
 
    text-align: center;
    position: absolute;
  
}

#countdown .labels li{
   
    color: #f47321;
    text-shadow: 1px 1px 0px #000;
    text-align: center;
    text-transform: uppercase;
    display: inline-block;
}
</style>