<template>
  <div>
    <nav class="teal">
      <div class="nav-wrapper container teal">
        <router-link to="/" class="brand-logo">IDPRAVUS</router-link>
        <ul id="nav-mobile" class="right hide-on-med-and-down">
          <li><router-link to="/board">게시판</router-link></li>
          <li><router-link to="/user">회원 목록</router-link></li>
          <li v-if="!loggedIn"><router-link to="/signup">회원 가입</router-link></li>
          <li v-if="!loggedIn"><a @click="login">로그인</a></li>
          <li v-if="loggedIn"><router-link to="/user/modify">정보수정</router-link></li>
          <li v-if="loggedIn"><a @click="logout">로그아웃</a></li>
        </ul>
      </div>
    </nav>
  </div>
</template>

<script>
import qstr from 'query-string';
import axios from 'axios';

export default {
  mounted() {
    const user = localStorage.user;
    if (user) {
      this.$store.dispatch('SET_USER', qstr.parse(user));
      this.getUserDetails();
    }
  },

  data: () => ({
    loggedIn: false,
    params: {
      response_type: 'token',
      client_id: 'vueclient',
      redirect_uri: `${process.env.VUE_APP_ORIGIN}/login?success`,
    },
    originHost: process.env.VUE_APP_ORIGIN,
  }),

  methods: {
    login() {
      const url = `${process.env.VUE_APP_API}/oauth/authorize?${qstr.stringify(this.params)}`;
      const options = 'width=600, height=600';

      const popup = window.open(url, 'auth', options);

      this.popupWatcher(popup, this.originHost).then((param) => {
        this.$store.dispatch('SET_USER', param);
        this.getUserDetails();
      });
    },

    popupWatcher(popup, exitUrl) {
      const parseUrl = document.createElement('a');
      parseUrl.href = exitUrl;

      return new Promise((resolve, reject) => {
        const polling = setInterval(() => {
          if (!popup || popup.closed || popup === undefined) {
            clearInterval(polling);
            reject(new Error('로그인 창 종료됨'));
          }

          try {
            if (popup.location.host === parseUrl.host) {
              const hash = qstr.parse(popup.location.hash.substring(1));
              if (hash.error) {
                reject(new Error(hash.error));
              } else {
                resolve(hash);
              }
              clearInterval(polling);
              popup.close();
            }
          } catch (error) {
            // cross origin frame exception
          }
        }, 250);
      });
    },

    getUserDetails() {
      const username = this.$store.getters.username;
      axios.get(`/api/v1/user/${username}`).then((res) => {
        if (res.status === 200) {
          this.$store.dispatch('SET_USER_DETAIL', res.data);
          this.loggedIn = true;
        }
      }).catch((err) => {
        if (err.response.status === 401) {
          this.logout();
        }
      });
    },

    logout() {
      this.loggedIn = false;
      this.$store.dispatch('SET_USER', null);
      this.$store.dispatch('SET_USER_DETAIL', null);
      this.$router.push('/');
    },
  },

};
</script>
