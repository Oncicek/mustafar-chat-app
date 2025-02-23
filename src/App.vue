<template>
  <div class="container main-container" v-if="isLoaded">
    <div class="row">
      <div class="col-5 sidebar-theme">
        <sidebar-component
          :people="sidebarData"
          :userNameOrig="userFullname"
          :favoritePeople="favoriteData"
        />
      </div>
      <div class="col" v-if="isEditing">
        <edit :people="peopleData" />
      </div>
      <div class="col chat-theme" v-if="!isEditing">
        <chat-component
          :user="user"
          :chatSide="chatSide"
          :favoritePeople="favoriteData"
        />
      </div>
      <div v-if="properties.showModal">
        <form
          action="
        "
        >
          <transition name="modal">
            <modal-component @close="properties.showModal = false">
              <template v-slot:header>
                <h3>Edit options</h3>
              </template>
              <template v-slot:body>
                <label> Display name: </label
                ><input type="text" v-model="user.displayName" />
                <label> Full name: </label
                ><input type="text" v-model="user.fullName" />
              </template>
              <template v-slot:footer>
                <div class="row">
                  <div class="col">
                    <button class="btn btn-danger" @click="cancelEditChanges()">
                      Cancel
                    </button>
                  </div>
                  <div class="col">
                    <button
                      class="btn btn-primary"
                      @click.prevent="saveEditChanges()"
                      type="submit"
                    >
                      OK
                    </button>
                  </div>
                </div>
              </template>
            </modal-component>
          </transition>
        </form>
      </div>
      <div v-if="login.isLogged < 0">
        <form @submit.prevent="loginToFirebase()">
          <transition name="modal">
            <modal-component :backColor="1">
              <template v-slot:header>
                <h3>Please log in:</h3>
              </template>
              <template v-slot:body>
                <label> E-mail: </label
                ><input type="text" class="login" v-model="login.email" />
                <label> Password: </label
                ><input
                  type="password"
                  class="login"
                  v-model="login.password"
                />
              </template>
              <template v-slot:footer>
                <div class="row">
                  <div class="alert alert-danger" v-if="login.isError">
                    {{ 'Login failed. Please try again' }}
                  </div>
                </div>
                <div v-if="login.isShowSpinner" class="lds-dual-ring"></div>
                <button v-else class="btn btn-primary" type="submit">
                  Login
                </button>
              </template>
            </modal-component>
          </transition>
        </form>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import SidebarComponent from './components/sidebar.vue'
import ChatComponent from './components/chat.vue'
import Edit from './components/edit.vue'
import ModalComponent from './components/shared/modal.vue'
import { reactive, onMounted, ref, inject, watch } from 'vue'
import axios from 'axios'
import firebase from './db'

export default {
  components: { ChatComponent, SidebarComponent, Edit, ModalComponent },
  setup() {
    const emitter: any = inject('emitter')
    const peopleData: any = ref([])
    let isEditing = ref(false)
    let userName = ref('')
    let userFullname = ref('')
    let userNameId = ref(0)
    const user: any = ref([])
    let isLoaded = ref(false)
    const sidebarData: any = ref([])
    const favoriteData: any = ref([])
    let userFavorite: any = ref([])
    let manAdded: any = ref([])
    const chatSide: any = ref([])
    let userFromFav = ref(-1)
    let showModal = ref(false)

    const fetchUsersData = async (forceFetch: boolean = false) => {
      if (cachedData() && !forceFetch) {
        login.isLogged = Number(localStorage.getItem('isLoggedIn'))
        showData(peopleData.value)
        isLoaded.value = true
        return
      }

      const response: any = await axios.get('../people.json')

      if (response.status === 200) {
        peopleData.value = response.data
        localStorage.setItem('fakeApi', JSON.stringify(peopleData.value))

        showData(peopleData.value)

        isLoaded.value = true
      }
    }

    const loginError = () => {
      login.isError = true

      setTimeout(() => {
        login.isError = false
      }, 5000)
    }

    const loginToFirebase = () => {
      login.isShowSpinner = true
      firebase
        .auth()
        .signInWithEmailAndPassword(login.email, login.password)
        .then(() => {
          login.isLogged = 1
          login.isShowSpinner = false
        })
        .catch(() => {
          loginError()
          login.isShowSpinner = false
        })
    }

    const saveEditChanges = () => {
      closeModal()
      showData(peopleData.value)
    }

    const cancelEditChanges = () => {
      closeModal()
    }

    const properties = reactive({
      showModal: showModal.value,
    })

    const login = reactive({
      isLogged: -1,
      email: '',
      password: '',
      isShowSpinner: false,
      isError: false,
    })

    const closeModal = () => {
      properties.showModal = false
    }

    const showData = (data: any) => {
      getUserData(data)
      getSidebarData(data)
      getFavoriteData(data)
    }

    const state = reactive({
      manAdded: manAdded.value,
      userFromFav: userFromFav.value,
    })

    const getSidebarData = (originalData: any) => {
      sidebarData.value = originalData.filter((x: any) => x.active === false)

      getChat(sidebarData.value)
    }

    const getUserData = (originalData: any) => {
      user.value = originalData.find((x: any) => x.active === true)

      userNameId.value = parseInt(user.value.id)
      userName.value = user.value.displayName
      userFullname.value = user.value.fullName
    }

    const getFavoriteData = (originalData: any) => {
      originalData.forEach((x: any) => {
        if (x.id === state.userFromFav) {
          for (let i = 0; i < state.manAdded.length; i++) {
            if (x.myFavorites.indexOf(parseInt(state.manAdded[i])) === -1) {
              x.myFavorites.push(parseInt(state.manAdded[i]))
            }
          }
        }
      })

      favoriteData.value = []
      userFavorite.value = user.value['myFavorites']

      userFavorite.value.forEach((x: number) => {
        favoriteData.value.push(
          originalData.find((y: any) => parseInt(y.id) === x)
        )
      })
    }

    const showEditComp = (isFromRow: boolean) => {
      if (isFromRow) {
        isEditing.value = !isFromRow
      } else {
        isEditing.value = !isEditing.value
      }
    }

    watch(
      () => login.isLogged,
      () => {
        localStorage.setItem('isLoggedIn', String(login.isLogged))
      }
    )

    const getChat = (originalData: any, chatId: number = 1) => {
      originalData.forEach((x: any) => {
        if (parseInt(x.id) == chatId) {
          chatSide.value = x
        }
      })
    }

    const updateFavorites = (chatIdParam: any) => {
      let userId = userNameId.value
      let chatId = parseInt(chatIdParam)

      peopleData.value.forEach((x: any) => {
        if (parseInt(x.id) === userId) {
          let indexInFavs = x.myFavorites.indexOf(chatId)
          let indexInPeople = favoriteData.value.findIndex(
            (y: any) => parseInt(y.id) === chatId
          )
          if (parseInt(indexInFavs) === -1) {
            x.myFavorites.push(chatId)
            favoriteData.value.push(
              peopleData.value.find(
                (person: any) => parseInt(person.id) == chatId
              )
            )
          } else {
            x.myFavorites.splice(indexInFavs, 1)
            favoriteData.value.splice(indexInPeople, 1)
          }
        }
      })

      localStorage.setItem('fakeApi', JSON.stringify(peopleData.value))
    }

    const cachedData = () => {
      if (localStorage.getItem('fakeApi')) {
        peopleData.value = JSON.parse(localStorage.getItem('fakeApi')!) || []
        return true //set false for debug
      } else {
        localStorage.setItem('fakeApi', JSON.stringify(peopleData.value))
        return false
      }
    }

    const clearFavsBeforeDestroyPerson = (fromId: number) => {
      let userId = userNameId.value

      peopleData.value.forEach((x: any) => {
        let indexInFavs = x.myFavorites.indexOf(fromId)
        let indexInPeople = favoriteData.value.findIndex(
          (y: any) => parseInt(y.id) === fromId
        )
        x.myFavorites.splice(indexInFavs, 1)
        favoriteData.value.splice(indexInPeople, 1)
      })
    }

    const destroyPerson = (fromId: number) => {
      clearFavsBeforeDestroyPerson(fromId)

      let index: number = peopleData.value.findIndex(
        (person: any) => parseInt(person.id) == fromId
      )
      peopleData.value.splice(index, 1)
      updateUserData(userNameId.value)
    }

    const updateUserData = (id: number) => {
      peopleData.value.forEach((x: any) => {
        if (x.id == id) {
          x.active = true
        } else {
          x.active = false
        }
      })

      if (peopleData.value) {
        userNameId.value = parseInt(
          peopleData.value.find((x: any) => x.active === true)?.id
        )

        getSidebarData(peopleData.value)
        getUserData(peopleData.value)
        getFavoriteData(peopleData.value)

        sidebarData.value = peopleData.value.filter(
          (x: any) => x.active === false
        )

        userName.value = peopleData.value.find(
          (x: any) => x.active === true
        )?.displayName

        userFullname.value = peopleData.value.find(
          (x: any) => x.active === true
        )?.fullName
      }
    }

    onMounted(() => {
      fetchUsersData()

      emitter.on('get-chat', (chat: any) => {
        getChat(sidebarData.value, chat.id)
      })

      emitter.on('show-edit-comp', (isFromRow: boolean) => {
        showEditComp(isFromRow)
      })

      emitter.on('update-user', (id: number) => {
        updateUserData(id)
        showData(peopleData.value)
      })

      emitter.on('fav-from-people', (fromId: number) => {
        updateFavorites(fromId)
      })

      emitter.on('destroy-from-people', (fromId: number) => {
        destroyPerson(fromId)
      })

      emitter.on('reset-config', () => {
        fetchUsersData(true)
        login.isLogged = -1
      })

      emitter.on('logout', () => {
        firebase
          .auth()
          .signOut()
          .then(() => {
            login.isLogged = -1
          })
      })

      emitter.on('close-modal', () => {
        properties.showModal = false
      })

      emitter.on('add-to-favorites', (chatId: any) => {
        updateFavorites(chatId)
      })

      emitter.on('show-modal', () => {
        properties.showModal = true
      })
    })

    return {
      showEditComp,
      isEditing,
      userNameId,
      userName,
      fetchUsersData,
      peopleData,
      sidebarData,
      favoriteData,
      getSidebarData,
      getFavoriteData,
      chatSide,
      userFavorite,
      user,
      state,
      isLoaded,
      updateFavorites,
      cachedData,
      destroyPerson,
      userFullname,
      properties,
      closeModal,
      saveEditChanges,
      cancelEditChanges,
      login,
      loginToFirebase,
      loginError,
      getChat,
      clearFavsBeforeDestroyPerson,
    }
  },
}
</script>

<style lang="scss">
* {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.main-container {
  border: 1px solid lightgray;
  box-shadow: 5px 5px 10px 1px rgba(100, 100, 100, 100);
  border-radius: 20px;
  margin: 20px 20px 0px 20px;
  max-width: 120vh !important;
}

.sidebar-theme {
  background-color: #f8eff1;
  border-radius: 20px 0px 0px 20px;
  min-height: 95vh;
}

.chat-theme {
  padding: 0px !important;
  border-radius: 0px 20px 20px 0px;
}

.lds-dual-ring {
  display: inline-block;
  width: 40px;
  height: 40px;
}
.lds-dual-ring:after {
  content: ' ';
  display: block;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: 6px solid rgb(0, 0, 0);
  border-color: #fff transparent rgb(4, 50, 255) transparent;
  animation: lds-dual-ring 0.75s linear infinite;
}
@keyframes lds-dual-ring {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.login {
  width: 100%;
  border-radius: 10px;
  font-size: 15px;
  padding: 5px 10px 5px 10px;
  border: 0.5px solid lightgray;
}

.login:focus {
  border: 0.5px solid lightgray;
  outline: 0;
}
</style>
