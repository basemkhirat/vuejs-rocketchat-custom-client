<template>
  <div class="chat">

    <div>Rocket.Chat websocket URL : {{webSocketUrl}}
    </div>
    <table>
      <tbody>
      <tr>

        <td>Username : <input id="login" type="text" v-model="username" :disabled="loggedIn"/></td>
        <td>Password : <input id="password" type="password" v-model="password"
                              :disabled="loggedIn"/></td>
        <td>
          <button v-on:click="loginBasic">Login</button>
        </td>
      </tr>

      <tr v-if="loggedIn">

        <td>Room <input readonly type="text" v-model="roomId"/></td>
        <td>
          <button v-on:click="connectRoom">Subscribe to room</button>
        </td>
      </tr>
      </tbody>
    </table>
    <div>
      <div v-if="roomConnected">
        Send message : <input type="text" v-on:keyup.enter="sendMessage" v-model="newMessage"/>
        <button v-on:click="sendMessage">Send</button>
      </div>
      <div id="messages" style="max-height: 400px;overflow-y: scroll">
        <div class="message" v-for="(message,messageIndex) in messages" :key="messageIndex">
          <b v-if="formatMessage (message).formattedDate">
            {{formatMessage (message).formattedDate}}</b>
          <code>{{message}}</code>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
  import rcApi from '../api'

  let api = null
  export default {
    name: 'Chat',
    data() {
      return {
        webSocketUrl: 'ws://63.32.52.233:3000/websocket',
        connectedToApi: true,
        loggedIn: false,
        userId: '',
        authToken: '',
        username: 'admin',
        password: 'qunEMIF09',
        roomName: 'sandbox',
        roomId: 'GENERAL',
        roomConnected: false,
        newMessage: '',
        messages: [],
        errors: [],
        lastSync: new Date ().getTime (),
        syncInterval: 30000
      }
    },
    mounted() {
      api = rcApi.connectToRocketChat (this.webSocketUrl)
      api.onError (error => this.errors.push (error))
      api.onCompletion (() => console.log ("finished"))
      api.onMessage (message => {
        this.messages.push (message)
      })


      api.connectToServer ()
        .subscribe (() => {
            api.keepAlive () // Ping Server

          },
          (error) => {
            this.errors.push (error)
          }
        )

      // vérification pour mobile devices
      setInterval (function () {
          let now = new Date ().getTime ()
          console.log ('verify sync')
          if ((now - this.lastSync) > this.syncInterval) {
            console.log ('out of sync')
            this.syncPage ()
          }
        }, 2000
      ) // vérifie toutes les 1 sec que 30 sec ont passé depuis la dernière synchro
    },
    methods: {
      decoWs() {
        let obj = api.webSocket
        for (var m in obj) {
          if (typeof obj[m] === "function") {
            console.log (m.toString ())
          }
        }
        api.webSocket.complete ()
      },
      recoWs() {
        api = rcApi.connectToRocketChat (this.webSocketUrl)
        api.onError (error => this.errors.push (error))
        api.onCompletion (() => console.log ("finished"))
        api.onMessage (message => {
          this.messages.push (message)
        })
        api.connectToServer ()
          .subscribe (() => {
              api.keepAlive () // Ping Server
              this.roomSubscribed = true;
            },
            (error) => {
              this.errors.push (error)
            }
          )
      },
      login() {
        if (!this.loggedIn) {
          api.loginWithAuthToken (this.authToken)
            .subscribe (apiEvent => {
              if (apiEvent.msg === 'result') {
                // success
                this.messages.push (apiEvent.msg)
                this.loggedIn = true
              }
            }, (error) => {
              this.errors.push (error)
            })
        }
      },
      loginBasic() {
        if (!this.loggedIn) {
          api.login (this.username, this.password)
            .subscribe (apiEvent => {
              if (apiEvent.msg === 'result') {
                // success
                this.messages.push (apiEvent.msg)

               if(apiEvent.error){
                 this.loggedIn = false;
               }else{
                 this.loggedIn = true;
               }
              }
            }, (error) => {
              alert("Sdf");
              this.errors.push (error)
            })
        }
      },
      connectRoom() {
        if (!this.roomConnected) {
          api.sendMessage ({
            "msg": "sub",
            "id": '' + new Date ().getTime (),
            "name": "stream-room-messages",
            "params": [
              this.roomId,
              false
            ]
          })
          this.roomConnected = true;
        }
      },
      sendMessage() {
        api.sendMessage ({
          "msg": "method",
          "method": "sendMessage",
          "id": '' + new Date ().getTime (),
          "params": [
            {
              "_id": '' + new Date ().getTime (),
              "rid": this.roomId,
              "msg": this.newMessage
            }
          ]
        })
        this.newMessage = ''
      },
      formatMessage(message) {
        let result = {message}
        if (message.fields !== undefined &&
          message.fields.args !== undefined &&
          message.fields.args.length > 0 &&
          message.fields.args[0].ts !== undefined &&
          message.fields.args[0].ts.$date !== undefined
        ) {
          result.formattedDate = new Date (message.fields.args[0].ts.$date)
        }
        return result
      },
      syncPage() {
        this.lastSync = new Date ().getTime ()
        console.log ('Synch')
        if (this.connectedToApi && api && api.webSocket !== null && api.webSocket.socket == null) {
          // on log et on redémarre la fenêtre
          console.log ('reload')
          window.location.reload ()
        }
      }
    }
  }
</script>

<style>
</style>
