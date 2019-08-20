<template>
  <v-app>
    <div v-if="pageView">
      <v-snackbar v-model="snackbar" :timeout="3000" top>
        <span v-html="snackbarMsg"></span>
        <v-btn color="pink" text @click="snackbar = false">승인</v-btn>
      </v-snackbar>

      <v-layout align-center justify-center row style="height: 100vh">
        <v-flex xs11 sm8 md6>
          <v-layout align-center>
            <v-img
              src="logo.png"
              lazy-src="logo.png"
              aspect-ratio="1"
              max-width="30"
              max-height="30"
              contain
            ></v-img>
            <h1 class="ml-3">URL Shortener</h1>
          </v-layout>
          <v-form v-model="form" v-if="!showResult" onsubmit="return false">
            <v-layout class="mt-4">
              <v-text-field
                v-model="origin"
                label="URL 입력"
                color="#00B8D4"
                :rules="rules"
                hint="단축된 URL은 공개되며 누구나 액세스할 수 있음"
                type="url"
                autofocus
                required
              ></v-text-field>
              <v-btn
                text
                color="#00B8D4"
                @click="(loader = 'loading'), shorten()"
                :disabled="loading || !form"
                :loading="loading"
                ><v-icon class="mr-1">mdi-flash</v-icon>URL 단축</v-btn
              >
            </v-layout>
          </v-form>

          <div v-if="showResult" class="mt-4">
            <v-alert
              class="elevation-3"
              prominent
              dark
              dense
              color="#455a64"
              icon="mdi-check"
            >
              <v-row align="center">
                <v-col class="grow">URL 단축 완료</v-col>
                <v-col class="shrink">
                  <v-btn
                    dark
                    color="#00B8D4"
                    @click="(showResult = false), (origin = '')"
                    ><v-icon class="mr-1">mdi-plus</v-icon>추가 단축</v-btn
                  >
                </v-col>
              </v-row>
            </v-alert>
            <v-layout>
              <v-text-field
                id="urlDisplay"
                color="#00B8D4"
                onclick="this.select();this.setSelectionRange(0,this.value.length)"
                :value="'https://l.edcan.kr/' + url"
                readonly
              ></v-text-field>
              <v-btn icon class="elevation-2" @click="copy()"
                ><v-icon v-text="'mdi-content-copy'"></v-icon
              ></v-btn>
            </v-layout>
          </div>
        </v-flex>
      </v-layout>
    </div>
  </v-app>
</template>

<script>
const dbRef = firebase.firestore().collection("URLs"),
  hashStr = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
export default {
  data: () => ({
    snackbar: false,
    snackbarMsg: "",
    loader: null,
    loading: false,
    origin: "",
    url: "",
    pageView: false,
    form: false,
    showResult: false,
    rules: [
      value => !!value || "필수 사항입니다.",
      value => {
        const pattern = /^(https:\/\/www\.|https:\/\/)?[a-z0-9]+([\-\.]{1}[a-z0-9]+)*\.[a-z]{2,5}(:[0-9]{1,5})?(\/.*)?$/g;
        return pattern.test(value) || "HTTPS 프로토콜의 URL만 단축 가능.";
      }
    ]
  }),

  created: function() {
    if (!!window.location.pathname && window.location.pathname !== "/") {
      dbRef
        .doc(window.location.pathname.substr(1))
        .get()
        .then(function(doc) {
          if (doc.exists) {
            location.replace(
              doc.data().origin.search("https://") === 0
                ? doc.data().origin
                : "https://" + doc.data().origin
            );
          } else {
            document.write("URL이 올바르지 않습니다.");
          }
        })
        .catch(function(error) {
          document.write("기다리십시오 ...");
          console.log("오류 : " + error);
          setTimeout(function() {
            location.reload();
          }, 2000);
        });
    } else {
      firebase.auth().signInAnonymously();
      this.pageView = true;
    }
  },

  methods: {
    snackbarShow(msg) {
      this.snackbar = true;
      this.snackbarMsg = msg;
    },
    shorten() {
      let v = this;
      let duplicates = 0;
      this.url = "";
      for (let i = 0; i < 4; i++) {
        this.url += hashStr.charAt(Math.floor(Math.random() * hashStr.length));
      }
      if (this.origin.search("https://") !== 0) {
        // 주소에 HTTPS 프로토콜이 없는 경우 URL 가공
        this.origin = "https://" + this.origin;
      }
      if(this.origin.search("l.edcan.kr") !== -1){
        // https://l.edcan.kr/* 금지
        this.snackbarShow("단축할 수 없는 URL입니다.");
        return;
      }
      dbRef
        .where("origin", "==", v.origin)
        .get()
        .then(querySnapshot => {
          // 중복 검사
          querySnapshot.forEach(doc => {
            v.url = doc.id;
            duplicates++;
          });
        });
      if (duplicates == 0) {
        // 중복이 없는 경우
        dbRef
          .doc(v.url)
          .get()
          .then(function(doc) {
            if (!doc.exists) {
              dbRef
                .doc(v.url)
                .set({
                  created: firebase.firestore.FieldValue.serverTimestamp(),
                  origin: v.origin
                })
                .then(function() {
                  v.showResult = true;
                  setTimeout(function() {
                    document.querySelector("#urlDisplay").focus();
                    document.querySelector("#urlDisplay").select();
                  }, 100);
                })
                .catch(function(error) {
                  v.snackbarShow("URL을 단축하지 못하였습니다.");
                  console.log("URL을 단축하지 못하였습니다: ", error);
                });
            } else {
              shorten();
            }
          })
          .catch(function(error) {
            v.snackbarShow("URL을 단축하지 못하였습니다.");
            console.log("URL을 단축하지 못하였습니다: ", error);
          });
      } else {
        // 중복이 없는 경우
        v.showResult = true;
        setTimeout(function() {
          document.querySelector("#urlDisplay").focus();
          document.querySelector("#urlDisplay").select();
        }, 100);
      }
    },
    copy() {
      let el = document.createElement("textarea");
      el.value = "https://l.edcan.kr/" + this.url;
      document.body.appendChild(el);
      el.select();
      document.execCommand("copy");
      if (el.value != "https://l.edcan.kr/" + this.url) {
        this.snackbarShow(
          "브라우저에서 클립보드 복사를 지원하지 않습니다.<br>생성된 URL을 수동으로 복사하여 주십시오."
        );
      } else {
        this.snackbarShow("복사되었습니다.");
      }
      document.body.removeChild(el);
    }
  },

  watch: {
    loader() {
      const l = this.loader;
      this[l] = !this[l];
      setTimeout(() => (this[l] = false), 3000);
      this.loader = null;
    }
  }
};
</script>