<template>
  <div class="container-flex chat-app">
    <div class="view chat">
      <div class="row chat-header">
        <div class="col-11 conv-name">
          <header>Conversation with {{ state.chatName }}</header>
        </div>
        <div class="col-1 fav-button">
          <button @click="addToFavorites(state.chatNameId)" id="fav-btn">
            <i v-if="state.isFavedBtn" class="bi-star"></i>
            <i v-else class="bi-star-fill" style="color: blue"></i>
          </button>
        </div>
      </div>
      <section class="chat-box">
        <div
          v-for="message in state.messages"
          :key="message.key"
          :class="
            message.userNameId == state.userNameId
              ? 'message current-user'
              : 'message'
          "
        >
          <div class="message-inner">
            <div class="username">
              {{ getChatNames(message.userNameId) }}
            </div>
            <div class="content">{{ message.content }}</div>
          </div>
        </div>
      </section>
      <footer>
        <div class="input-wrapper">
          <form @submit.prevent="sendMessage">
            <input
              type="text"
              v-model="inputMessage"
              :placeholder="'Message ' + state.chatName"
            />
            <button type="submit" class="submit-btn">
              <i class="bi-arrow-up-circle-fill" style="color: lightgray"></i>
            </button>
          </form>
        </div>
      </footer>
    </div>
  </div>
</template>

<script lang="ts">
import { reactive, onMounted, ref, inject, onUnmounted } from 'vue'
import db from '../db'

export default {
  props: {
    user: {
      type: Object,
    },
    chatSide: {
      type: Object,
    },
    favoritePeople: {
      type: Object,
    },
  },
  setup(props: any) {
    const emitter: any = inject('emitter')
    const inputMessage = ref('')
    const conversationId = ref(0)
    let favoriteData: any = ref([])
    let user: any = ref([])
    let isFavedBtn = ref()

    const state = reactive({
      chatName: props.chatSide['displayName'],
      chatNameId: props.chatSide['id'],
      username: props.user['displayName'],
      userNameId: props.user['id'],
      conversationId: conversationId,
      isFavedBtn: isFavedBtn.value,
      favPeople: props.favoritePeople,
      messages: [],
      user: user,
    })

    const sendMessage = () => {
      getConversationId()
      const messageRef = db.database().ref('message')
      if (inputMessage.value === '' || inputMessage.value === null) {
        return
      }

      const message = {
        userNameId: state.userNameId,
        username: state.username,
        content: inputMessage.value,
        timestamp: Date.now(),
        conversationId: state.conversationId,
      }

      messageRef.push(message)
      inputMessage.value = ''
    }

    const getConversationId = () => {
      state.conversationId = calcConvId(state.userNameId, state.chatNameId)
    }

    emitter.on('get-chat', (chat: any) => {
      state.chatNameId = chat.id
      state.chatName = chat.chatName

      getConversationId()
      getFreshData(state.conversationId)
    })

    const messageRef = db.database().ref('message')

    const getFavoriteMessage = (favoritePeople: any, messages: any) => {
      favoriteData.value = []

      for (let i = 0; i < favoritePeople.length; i++) {
        let favs: any = {
          convId: calcConvId(state.userNameId, favoritePeople[i].id) as number,
          favoritePersonId: favoritePeople[i].id as number,
          lastMessage: '',
        }

        favs.lastMessage = getLastMessage(messages, favs)
        favoriteData.value.push(favs)
      }

      emitter.emit('fav-message', favoriteData.value)
    }

    const calcConvId = (userNameId: string, chatNameId: any) => {
      return (parseInt(userNameId) + 1) * (parseInt(chatNameId) + 1)
    }

    const getLastMessage = (messages: any, favs: any) => {
      for (let j = messages.length - 1; j >= 0; j--) {
        if (messages[j].userNameId) {
          if (parseInt(messages[j].conversationId) === parseInt(favs.convId)) {
            console.log(messages[j].content)
            return messages[j].content
          }
        }
      }
    }

    const addToFavorites = (chatId: number) => {
      emitter.emit('add-to-favorites', chatId)
      emitter.emit('fav-message', favoriteData.value)
      getIsFavedBtn()
    }

    emitter.on('update-user-favs', () => {
      emitter.emit('fav-message', favoriteData.value)
    })

    const getIsFavedBtn = () => {
      let isFaved = Object.values(state.favPeople).findIndex(
        (x: any) => x.id === state.chatNameId
      )

      if (isFaved > -1) {
        state.isFavedBtn = false
      } else {
        state.isFavedBtn = true
      }
    }

    const getFreshData = (conversationId: number) => {
      messageRef.on('value', (snapshot) => {
        const data = snapshot.val()
        let messages: any = []

        Object.keys(data).forEach((key) => {
          messages.push({
            id: key,
            userNameId: data[key].userNameId,
            username: data[key].username,
            content: data[key].content,
            timestamp: data[key].timestamp,
            conversationId: data[key].conversationId,
          })
        })

        let clearedMsg = messages.filter(
          (x: any) => x.conversationId === conversationId
        )

        clearedMsg.reverse()

        getFavoriteMessage(props.favoritePeople, messages)
        getIsFavedBtn()
        state.messages = clearedMsg
      })
    }

    const getChatNames = (usernameIdParam: number) => {
      switch (usernameIdParam) {
        case state.chatNameId:
          return state.chatName
        case state.userNameId:
          return state.username
        default:
          return 'missing id, please contact your IT administrator'
      }
    }

    onMounted(() => {
      getIsFavedBtn()
      getConversationId()
      getFreshData(state.conversationId)
    })

    return {
      state,
      inputMessage,
      sendMessage,
      getFreshData,
      props,
      calcConvId,
      favoriteData,
      addToFavorites,
      getIsFavedBtn,
      getChatNames,
    }
  },
}
</script>

<style scoped lang="scss">
#fav-btn {
  border: 0px;
  background-color: white;
}

.submit-btn {
  background-color: white;
  border: 0px;
  border-radius: 100px;
  font-size: 25px !important;
  margin-right: 10px;
}

.chat-header {
  padding: 15px !important;
  --bs-gutter-x: 0 !important;
  padding: 0px 10px;
  background-color: white;
  font-size: 15px;
  border-radius: 0px 20px 0px 0px;
}

.fav-button {
  text-align: right;
  align-content: right;
}

.conv-name {
  text-align: center;
}

.input-wrapper {
  border: 1px solid gray;
  border-radius: 20px;
}

.view {
  display: flex;
  justify-content: center;
  max-height: 100vh;
  &.login {
    align-items: center;
    .login-form {
      display: block;
      width: 100%;
      padding: 15px;
    }
  }
  &.chat {
    flex-direction: column;
    padding-bottom: 4vh;
    .chat-box {
      overflow-y: auto;
      flex-direction: column-reverse;
      display: flex;
      max-height: 80vh;
      min-height: 80vh;
      background-color: #fff;
      box-shadow: 0px 0px 12px rgba(100, 100, 100, 0.2);
      flex: 1 1 100%;
      padding: 30px;
      .message {
        display: flex;
        margin-bottom: 15px;

        .message-inner {
          .username {
            color: #888;
            font-size: 12px;
            margin-bottom: 5px;
            padding-left: 5px;
            padding-right: 5px;
          }
          .content {
            word-break: break-word;
            display: inline-block;
            padding: 10px 20px;
            background-color: dodgerblue;
            border-radius: 50px;
            color: #fff;
            font-size: 15px;
            line-height: 1em;
            text-align: left;
          }
        }
        &.current-user {
          margin-top: 10px;
          justify-content: flex-end;
          text-align: right;
          .message-inner {
            max-width: 75%;
            .content {
              color: black;
              background-color: lightgray;
            }
          }
        }
      }
    }
    footer {
      position: sticky;
      bottom: 0px;
      padding: 0px 20px;
      form {
        display: flex;
        box-shadow: 0px 0px 12px rgba(100, 100, 100, 0.2);
        border-radius: 20px;
        input[type='text'] {
          border: 0px;
          border-radius: 20px;
          flex: 1 1 100%;
          width: 100%;
          padding: 5px 10px 1px 10px;
          margin: 0px 2px 5px 2px;
          font-size: 15px;
          font-weight: 100;
          &::placeholder {
            color: darkgray;
          }
        }
        input:focus {
          outline: none;
        }

        input[type='submit'] {
          display: block;
          background-color: lightgray;
          color: #fff;
          font-size: 15px;
          font-weight: 700;
        }
      }
    }
  }
}
</style>
